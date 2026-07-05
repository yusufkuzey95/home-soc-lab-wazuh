# Detection 2 - New user account created

**MITRE ATT&CK:** T1136.001 (Create Account: Local Account), tactic Persistence
**Wazuh rule:** 5902 (level 8), new user - plus 5901 (level 8) for the group
**Log source:** /var/log/auth.log on the Ubuntu endpoint

## What the attack is

This is what an attacker does right after they get in. A stolen password is fragile, it can be changed or the session drops. So they create their own account to come back to later. If the blue team only disables the account that got popped, the attacker still has their backdoor account and is still inside. That's why this maps to Persistence, not Initial Access.

It ties into detection 1 as a mini kill chain: brute force to get in, then create an account to stay in.

## How I simulated it

One command as root:

```bash
sudo useradd -m backdoor
```

Then cleaned it up after with `sudo userdel -r backdoor` so I'm not leaving a real backdoor account sitting on the box.

## How Wazuh caught it

Two alerts from that one command:

```
00:28:14  5902  LEVEL 8  New user added to the system
00:28:14  5901  LEVEL 8  New group added to the system
```

5902 is the one I cared about, it's mapped to T1136. 5901 fired too because useradd also creates a group named backdoor, so one action left two artifacts in the log. Same thing I saw in detection 1 where one login failure showed up as both an sshd and a PAM event.

This is a stateless rule, it matches a single log line ("new user...") and fires. No counting, no time window, unlike the brute force rule.

## Level 8 vs level 5, and why

The failed logins in detection 1 were level 5. This single account creation is level 8, higher, even though the rule is simpler. Wazuh sets severity by how much an analyst should care, not by how hard the detection was. A fumbled password is background noise. A new account showing up is rare and is a classic persistence move, so it deserves more attention by default.

## What a false positive looks like

Account creation is completely normal in IT. Onboarding a new employee, a package installer creating a service account, a config management tool (Ansible etc) provisioning users. All of those fire this exact alert. The log line for a legit new hire and an attacker backdoor is identical. What tells them apart is context: was this account creation expected, was it done by an admin during business hours through the normal process, or did it show up at 3am right after a burst of failed logins. The alert's job isn't to decide, it's to put it in front of a human who can ask "was this supposed to happen".
