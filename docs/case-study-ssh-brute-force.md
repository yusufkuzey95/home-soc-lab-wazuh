# Case study: catching an SSH brute force end to end

This is one attack walked all the way through, from me running it to the alert showing up in Wazuh, plus how I'd actually triage it and where the detection falls short. I picked SSH brute force because it's the most common thing you'll see hitting any box with port 22 open, and it's a clean example of the one thing a SIEM does that grep can't: correlate a bunch of small events into one.

## The technique

SSH brute force is MITRE ATT&CK T1110 (Brute Force), tactic Credential Access. Someone points a tool at your SSH service and tries password after password hoping one lands. It's noisy and old but it never stopped working, because there's always a box somewhere with a weak password and 22 open to the internet. Automated bots do this constantly.

The thing that makes it detectable is also what makes it an attack: volume. One person mistyping their password once is nothing. The same source firing off dozens of attempts in a minute is not a person, it's a script.

## What I simulated

From inside my Ubuntu endpoint I ran 10 SSH logins as a user that doesn't exist ("hacker"), with a wrong password, against localhost:

```bash
for i in $(seq 1 10); do sshpass -p wrongpass ssh -o StrictHostKeyChecking=no -o PreferredAuthentications=password -o ConnectTimeout=2 hacker@localhost true; done
```

Every attempt fails on purpose. I used sshpass because ssh won't take a password from a script, it insists on reading the keyboard, which is a small security feature that tripped me up the first time (my first version just hung at the password prompt).

## How it looked in the raw logs

Each failed attempt makes sshd write a line to /var/log/auth.log. These are the actual lines from my run:

```
Jul 04 23:27:55 ubuntu-host sshd[6014]: Failed password for invalid user hacker from 127.0.0.1 port 57816 ssh2
Jul 04 23:28:11 ubuntu-host sshd[6038]: Invalid user hacker from 127.0.0.1 port 56646
Jul 04 23:28:13 ubuntu-host sshd[6038]: Failed password for invalid user hacker from 127.0.0.1 port 56646 ssh2
```

On their own these are just text sitting in a file on the endpoint. Nobody is watching them there. The Wazuh agent tails that file and ships each new line to the manager over port 1514.

## How the rule caught it

The manager decodes each line with the sshd decoder, pulling out the fields it cares about: the source IP (127.0.0.1), the user it targeted (hacker), the program (sshd). Then two rules do the work:

Rule 5710, level 5, fires on a single "invalid user" line. It's mapped to T1110.001 (Password Guessing). On its own it's basically informational, one failed login is noise.

Rule 5712, level 10, is the real detection. It doesn't read the log directly, it watches for rule 5710 firing 8 times from the same source IP inside 120 seconds. This is the part grep can't do, it's counting related events across time and keying them to one source. When the count hits 8 it fires and maps to T1110 (Brute Force).

Here's the actual chain from my run. Each 5710 is one failed login, and the 8th one trips 5712:

```
23:27:45  5710  (1) invalid user           lvl 5
23:27:47  5710  (2)                          lvl 5
23:27:49  5710  (3)                          lvl 5
23:27:51  5710  (4)                          lvl 5
23:27:51  5710  (5)                          lvl 5
23:27:53  5710  (6)                          lvl 5
23:27:55  5710  (7)                          lvl 5
23:27:57  5710  (8)                          lvl 5
23:27:57  5712  brute force                  LEVEL 10   <-- fires the same second as the 8th
```

The failures kept coming after that but no second level-10 fired, because the rule has ignore="60" which mutes duplicates for a minute so one attack doesn't flood the queue.

The final alert in the indexer looked like this (trimmed):

```
rule.id:          5712
rule.level:       10
rule.description: sshd: brute force trying to get access to the system. Non existent user.
rule.frequency:   8
data.srcip:       127.0.0.1
rule.mitre.id:    ["T1110"]
rule.mitre.tactic:["Credential Access"]
```

One detail I didn't expect: every failed login actually showed up twice, once as the sshd rule (5710) and once as a PAM rule (5503, "PAM: User login failed"). Same real-world action, seen by two different logging systems on the box. Good reminder that one event often lands in more than one place, and part of the analyst's job is realizing they're the same thing.

## How I'd triage this as an analyst

If this popped in my queue, the level 10 gets my attention first. The questions I'd ask, in order:

1. Source IP. 127.0.0.1 here because I attacked locally, but a real one would be external. Is it a known IP or brand new? Geolocation? Is it hitting other hosts too (that's where the central manager matters, a single endpoint can't see that)?
2. Target user. "hacker" doesn't exist, which actually tells me this is dumb automated guessing, not someone who knows my usernames. Attempts against a real account like "root" or a known admin would worry me more.
3. Did anything succeed. This is the big one. A pile of failures followed by a successful login from the same IP means they got in, and that's no longer a brute force alert, that's an incident. Wazuh has a separate rule for exactly that pattern.

## What a false positive looks like

This rule fires on 8 failures from one IP in 2 minutes. Plenty of harmless things do exactly that:

- a cron job or backup script with an expired password, retrying in a loop
- a user whose saved SSH credential went stale after a password change, and their client keeps auto-retrying
- a monitoring or health check pointed at the wrong port or user
- someone on the team fat-fingering their password a bunch of times, then getting it right

At the log level every one of these looks identical to an attack, because the rule sees behavior, not intent. The way you tell them apart is context: is the source a known internal IP or a service account, is the targeted user real, did it happen during a change window, and did anything eventually succeed. The alert's job isn't to decide, it's to surface the pattern so a human can ask those questions. Tuning this in the real world usually means whitelisting known service-account IPs rather than turning the rule down, because you still want to catch the real thing.

## Where this detection falls short

The rule keys on 8 failures from one IP in 120 seconds. An attacker who knows that can slip under it:

- stay at 7 attempts and stop
- spread the attempts out over more than 120 seconds (low and slow), so the counter keeps resetting
- come from many different IPs (a botnet), so no single source ever hits 8

Any threshold detection has a threshold someone can duck below. That's not a reason to drop it, it still catches the loud automated majority, which is most of what's actually out there. But it's why a real SOC layers other detections on top: a slower rule that counts failures over a whole day, and a rule that alerts on a successful login right after a burst of failures, which catches the case where they actually guessed right. No single rule is the whole answer, and knowing that is kind of the point.
