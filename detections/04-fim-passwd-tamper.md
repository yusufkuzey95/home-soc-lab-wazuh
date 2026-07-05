# Detection 4 - File tampering caught by FIM

**MITRE ATT&CK:** T1136 / T1078 (rogue account), tactic Persistence / Privilege Escalation
**Wazuh rule:** 550 (level 7), integrity checksum changed
**Capability:** File Integrity Monitoring (syscheck), not a log rule

## Why this one is different

Detections 1-3 all read logs. Something has to write a log line for the rule to see it. This one is about what happens when the attacker does something that DOESN'T produce a nice log.

Instead of running useradd (which fires detection 2), the attacker just edits /etc/passwd directly and adds a root account by hand. No useradd, no syslog event, so rule 5902 never fires. The log-based detection is blind to it. Same tool-vs-outcome problem I hit in detection 3.

File Integrity Monitoring is the answer. The agent keeps a checksum of important files and alerts when the contents change, no matter how the change happened. It doesn't care if you used useradd, vim, echo or a script. It just notices the file isn't what it was.

## Setup

Default FIM watches /etc but only scans every 12 hours, way too slow to catch anything live. So I turned on realtime monitoring for /etc in the agent config:

```xml
<directories realtime="yes" report_changes="yes" check_all="yes">/etc</directories>
```

realtime="yes" uses inotify to catch changes the instant they happen. report_changes="yes" is the important part, it makes the alert include the actual diff of what changed. Restarted the agent and confirmed "Real-time file integrity monitoring started" in the log.

## How I simulated it

Added a root account by editing the file directly, no useradd:

```bash
echo 'hacker:x:0:0:backdoor root account:/root:/bin/bash' | sudo tee -a /etc/passwd
```

UID 0 is what makes it root. Cleaned it back out after with sed.

## How Wazuh caught it

One alert, and notice which one it is NOT:

```
02:15:55  rule 550  LEVEL 7  Integrity checksum changed  /etc/passwd
   diff:  30a31 > hacker:x:0:0:backdoor root account:/root:/bin/bash
```

- Detection 2's rule (5902) did NOT fire, because there was no useradd log. Confirmed the blind spot.
- FIM caught it as rule 550.
- Because of report_changes, the alert shows the exact line that got added. An analyst sees 0:0 and immediately knows someone slipped in a root account by hand. That diff is what turns "a file changed" into "here is the attack".

## The lesson

Log rules watch HOW something happened. FIM watches WHAT changed. An attacker who is careful about not generating logs still can't hide that the file itself is different. That's why FIM on critical files (/etc/passwd, /etc/sudoers, ssh keys, web roots) is often the only thing that catches a quiet attacker.

## What a false positive looks like

FIM is noisy by nature. A system update, a package install, an admin editing a config, all of them change files in /etc and fire rule 550. Watching /etc in realtime will absolutely produce false positives during normal patching. The tuning move is to scope FIM to the files that really matter and use ignore rules for the paths that change constantly (the config already ignores things like /etc/mtab and .log files for this reason). The report_changes diff is what lets an analyst quickly tell "this was apt updating a config" from "someone added a UID 0 account".
