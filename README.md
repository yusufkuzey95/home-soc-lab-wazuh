# Home SOC Lab (Wazuh)

My third security project. I built a small SOC setup at home. The setup is Wazuh running in Docker, one Ubuntu endpoint sending its logs in, and a few detections that I test by attacking my own endpoint (safely) and checking that the alert actually fires.

Everything here works and is validated. There are 4 detections mapped to MITRE ATT&CK, one of them a custom rule I wrote to close a gap I found in Wazuh's default ruleset, plus a full case study on one of them.

## Why

Pretty much every SOC analyst posting asks for SIEM experience. I did a phishing triage CLI and a Sigma detection rule pack before this, so I knew what detection rules look like on paper. This one is about the engine itself: getting a real SIEM up and watching it catch attacks as they happen. I went with Wazuh because it's free, open source, and real companies actually run it in production.

## How a log gets from the endpoint to my screen

```
something happens on the Ubuntu endpoint (failed ssh login, new user, etc)
    |
    v
a line gets written to /var/log/auth.log like normal
    |
    v
the wazuh agent on the endpoint picks up the new line
and sends it to the manager (tcp 1514, encrypted)
    |
    v
the manager (docker container) parses it into fields
and runs it through its rules
    |
    v
if a rule matches -> alert -> stored in the indexer (the database)
    |
    v
I see it in the dashboard in my browser at https://localhost
```

The manager is where detection actually happens. It can also correlate over time, so one failed login is just an event, but 10 in two minutes from the same IP turns into a brute force alert. The indexer is basically the database and the dashboard is just the UI that queries it.

## Detections

4 detections, each one the same way: run a safe simulation against my endpoint, confirm the alert fires in Wazuh, write up what I saw and what a false positive would look like.

| # | Technique | MITRE | Wazuh rule | Write-up |
|---|-----------|-------|------------|----------|
| 1 | SSH brute force | T1110 | 5712 (lvl 10) | [01-ssh-brute-force.md](detections/01-ssh-brute-force.md) |
| 2 | New user account created | T1136.001 | 5902 (lvl 8) | [02-new-user-account.md](detections/02-new-user-account.md) |
| 3 | Privilege escalation (usermod to sudo) | T1548.003 | **100002 (custom, lvl 10)** | [03-sudo-privilege-escalation.md](detections/03-sudo-privilege-escalation.md) |
| 4 | File tampering (/etc/passwd, direct edit) | T1136 / T1078 | 550 (FIM, lvl 7) | [04-fim-passwd-tamper.md](detections/04-fim-passwd-tamper.md) |

These aren't random, they line up as a little kill chain: brute force your way in (1), create an account so you can get back in (2), give that account root (3), and if you want to skip the noisy tools, just edit /etc/passwd directly (4). Detection 4 is the interesting one, because editing the file by hand slips right past the log-based rule in detection 2, and only file integrity monitoring catches it. Same lesson as detection 3: watch what actually changed, not the specific tool that did it.

## Case study

One technique walked through end to end: what I simulated, what the raw logs looked like, how the rule caught it, how I'd triage it, and what a false positive looks like. I picked SSH brute force because it's the clearest example of the one thing a SIEM does that grep can't, correlate a pile of small events into one.

Read it here: [docs/case-study-ssh-brute-force.md](docs/case-study-ssh-brute-force.md)

## How to reproduce

Rough steps if you want to stand this up yourself. Assumes Windows with Docker Desktop and WSL2.

1. Clone Wazuh's official docker repo and use the single-node setup:
   ```
   git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.6
   cd wazuh-docker/single-node
   ```
2. Set the kernel setting the indexer needs (inside the WSL VM), or it will crash-loop:
   ```
   sysctl -w vm.max_map_count=262144
   ```
3. Generate the certs, then bring it up:
   ```
   docker compose -f generate-indexer-certs.yml run --rm generator
   docker compose up -d
   ```
4. Log into the dashboard at https://localhost (self-signed cert warning is expected), default admin / SecretPassword.
5. On the endpoint (I used Ubuntu 24.04 on WSL2), add Wazuh's apt repo and install the agent pointed at the manager:
   ```
   WAZUH_MANAGER="localhost" apt-get install wazuh-agent
   systemctl enable --now wazuh-agent
   ```
   Confirm it shows up as Active under Endpoints in the dashboard.
6. My custom privilege-escalation rule is in [wazuh-config/local_rules.xml](wazuh-config/local_rules.xml), drop it into the manager's `/var/ossec/etc/rules/local_rules.xml` and restart the manager.
7. Each detection write-up in [detections/](detections/) has the exact command I ran to trigger it.
