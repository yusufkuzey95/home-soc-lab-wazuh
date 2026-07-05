# Detection 1 - SSH brute force

**MITRE ATT&CK:** T1110 (Brute Force), child T1110.001 (Password Guessing)
**Wazuh rule:** 5712 (level 10), built on 5710 (level 5)
**Log source:** /var/log/auth.log on the Ubuntu endpoint (sshd + PAM)

## What the attack is

Someone hammers the SSH service trying passwords until one works. It's noisy and old but it never went away, because it still works against weak passwords and exposed servers. The whole point of catching it is that it's usually automated, so it comes in fast bursts you can spot.

## How I simulated it

From inside the endpoint, ran 10 SSH logins as a user that doesn't exist ("hacker") with a wrong password, all against localhost:

```bash
for i in $(seq 1 10); do sshpass -p wrongpass ssh -o StrictHostKeyChecking=no -o PreferredAuthentications=password -o ConnectTimeout=2 hacker@localhost true; done
```

Each attempt fails on purpose. I needed sshpass because ssh won't take a password from a script, it insists on reading the keyboard, so the first version of this just hung at the prompt.

## How Wazuh caught it

Two rules working together:

- **5710** fires on a single "invalid user" line from sshd. Level 5, basically informational on its own.
- **5712** is the real detection. It doesn't read logs directly, it watches for 5710 firing 8 times from the same source IP inside 120 seconds. When that happens it fires at level 10 and maps to T1110.

Here's the actual chain from my run, the 8th failure triggers it:

```
23:27:45  5710  (1) invalid user
23:27:47  5710  (2)
23:27:49  5710  (3)
23:27:51  5710  (4)
23:27:51  5710  (5)
23:27:53  5710  (6)
23:27:55  5710  (7)
23:27:57  5710  (8)   <- eighth failure
23:27:57  5712  LEVEL 10  brute force
```

The failures kept coming after that but no second level-10 alert fired, because the rule has ignore="60" which mutes duplicates for a minute so one attack doesn't spam the queue.

One thing I didn't expect: every failed login showed up twice, once as 5710 (from sshd's own log) and once as 5503 (PAM, the OS-wide auth layer). Same action, two different log sources. Good reminder that the same event often lands in more than one place.

## What a false positive looks like

This rule keys on 8 failures from one IP in 2 minutes. Plenty of harmless things do that:

- a cron job or backup script with an expired password retrying in a loop
- a user whose saved credential went stale after a password change, and their client keeps auto-retrying
- a monitoring/health check pointed at the wrong port or user

All of those look identical to an attack at the log level. That's the core problem, the rule sees behavior, not intent. The way you tell them apart is context, is this a known internal IP, is it a service account, did any attempt eventually succeed.

## Limits of this detection

If an attacker knows the threshold they can slip under it: stay at 7 tries, spread the attempts over more than 120 seconds (low and slow), or come from lots of different IPs so no single one hits 8. A threshold detection always has a threshold someone can duck below. That's not a reason to drop it, it catches the loud automated stuff, but it's why a real SOC layers a slower long-window rule and a "success right after a pile of failures" rule on top.
