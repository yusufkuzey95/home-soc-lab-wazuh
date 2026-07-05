# Detection 3 - Privilege escalation via usermod (custom rule)

**MITRE ATT&CK:** T1548.003 (Abuse Elevation Control Mechanism: Sudo), tactic Privilege Escalation
**Wazuh rule:** 100002 (level 10) - I wrote this one
**Log source:** /var/log/auth.log on the Ubuntu endpoint

## What the attack is

Next step in the kill chain. The attacker has a backdoor account (detection 2) but it's just a normal user, it can't do much. So they add it to the sudo group, which on Linux is what gives you the right to run commands as root. One command and their nobody account becomes an admin account.

## How I simulated it

```bash
sudo useradd -m eviluser && sudo usermod -aG sudo eviluser
```

The second command is the actual escalation, it adds eviluser to the sudo group. Cleaned up after with userdel.

## The interesting part - the built-in rule missed it

When I first ran this, I expected a loud privilege escalation alert. Instead the escalation only showed up as a generic "Successful sudo to ROOT" at level 3, same low priority as any normal sudo command. No dedicated alert for the group change.

I dug into why. Wazuh DOES ship a rule for this (2961, "User added to group sudo"), but it chains off rule 2960 which has `<decoded_as>gpasswd</decoded_as>`. That means it only fires when the group change comes from the `gpasswd` program. I used `usermod -aG`, which writes a different log line from a different program, so it sails right past the built-in rule.

I confirmed the gap with wazuh-logtest, feeding it my exact log line:

```
usermod[6787]: add 'eviluser' to group 'sudo'
   -> Phase 1: program_name: 'usermod'
   -> Phase 2: No decoder matched
   -> (no rule fired)
```

Same outcome as gpasswd (user gets root), different program writes the log, so the vendor rule is blind to it. That's a real detection gap.

## The custom rule I wrote

Lives in local_rules.xml (see [wazuh-config/local_rules.xml](../wazuh-config/local_rules.xml)):

```xml
<rule id="100002" level="10">
  <program_name>usermod</program_name>
  <match>to group 'sudo'</match>
  <description>User added to sudo group via usermod - possible privilege escalation.</description>
  <mitre>
    <id>T1548.003</id>
  </mitre>
</rule>
```

Why it works even though "no decoder matched": the pre-decode phase still pulls out program_name before any decoder runs, so I match on program_name=usermod plus the literal text "to group 'sudo'". Custom rule IDs have to be 100000+ so they don't collide with built-in rules. Level 10 because granting root is worth an analyst's attention right away.

## Before and after

- Before my rule: escalation = rule 5402, level 3, buried in normal sudo noise.
- After my rule: same action also fires rule 100002, level 10, mapped to T1548.003, right at the top of the queue.

Validated end to end: created a fresh user, added it to sudo, and watched 100002 land in the indexer at level 10.

## What a false positive looks like

An admin legitimately granting someone sudo does the exact same thing and fires this exact rule. That's fine, this is a rule you WANT to be noisy, because every sudo-group addition should be reviewed. The move here isn't to make it quieter, it's to pair it with context: was this done by an admin, through the ticketed process, or did it show up right after a brute force and a new account got created (which is exactly the chain in detections 1 through 3).
