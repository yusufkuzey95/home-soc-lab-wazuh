# Home SOC Lab (Wazuh)

My third security project. I built a small SOC setup at home so I can say I've actually used a SIEM instead of just reading about one. The setup is Wazuh running in Docker, one Ubuntu endpoint sending its logs in, and a few detections that I test by attacking my own endpoint (safely) and checking that the alert actually fires.

Still a work in progress. I'm committing as I go, so some sections below are empty for now.

## Why

Pretty much every SOC analyst posting asks for SIEM experience. I did a phishing triage CLI and a Sigma detection rule pack before this, so I knew what detection rules look like on paper, but I'd never had the actual engine running anywhere. This project is me fixing that. I went with Wazuh because it's free, open source, and real companies actually run it in production.

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

The plan is 3-4 detections mapped to MITRE ATT&CK. Same process for each one: run a safe simulation against my endpoint, confirm the alert fires in Wazuh, write up what I saw.

| # | Technique | MITRE | Wazuh rule | Write-up |
|---|-----------|-------|------------|----------|
| 1 | SSH brute force | T1110 | 5712 (lvl 10) | [01-ssh-brute-force.md](detections/01-ssh-brute-force.md) |
| 2 | New user account created | T1136.001 | 5902 (lvl 8) | [02-new-user-account.md](detections/02-new-user-account.md) |

## Case study

One technique walked through end to end: what I simulated, what the raw logs looked like, how the rule caught it, and what a false positive on that same rule would look like. Coming after the detections are done.

## What I learned

Filling this in as I go, the honest version including what broke and how I figured it out.

## How to reproduce

Coming at the end, once I'm sure the steps I'd write down are actually the right ones.
