# Home SOC Lab (Wazuh)

I'm building a small SOC environment at home to actually understand how a SIEM works, not just read about one. The setup: Wazuh running on Docker, one Linux endpoint shipping its logs in, and a handful of detections that I validate by attacking my own endpoint (safely) and watching the alerts fire.

This is my third security project, after a phishing triage CLI and a Sigma detection-rule pack. Those taught me how detection *rules* work. This one is about the engine they run in.

> **Status: work in progress.** I'm building this over a couple of days and committing as I go, so some sections below are still stubs.

## Why I built this

Every SOC analyst job posting mentions SIEM experience, and you can't get that from a textbook. I wanted to be able to say in an interview: I stood up the SIEM myself, enrolled the agent myself, broke it, fixed it, and watched real alerts fire from attacks I ran. Wazuh made sense because it's free, open source, and actually used in production at companies that don't have Splunk budgets.

## Architecture

Logs flow from the endpoint to my browser like this:

```
[Ubuntu endpoint]                        [Wazuh server - Docker, single node]
  something happens                        ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
  (failed login, file change)              │   manager    │   │   indexer   │   │  dashboard  │
        │                                  │ decodes logs │──▶│  stores     │──▶│  where I    │
        ▼                                  │ + runs rules │   │  alerts     │   │ investigate │
  written to /var/log/...                  └─────────────┘   └─────────────┘   └─────────────┘
        │                                         ▲
        ▼                                         │
  [Wazuh agent] ── encrypted, TCP 1514 ───────────┘
  tails the logs, ships new events
```

In plain English: the endpoint writes logs like it always does. The Wazuh agent (a small program on the endpoint) tails those logs and ships every new event to the manager. The manager parses each event into structured fields and runs it through its ruleset — including rules that correlate patterns over time, like a burst of failed logins. Anything that matches becomes an alert, gets stored in the indexer, and shows up in the dashboard where I can search and investigate it.

## Detections

Each detection gets validated the same way: run a safe simulation against my endpoint, confirm the alert fires in Wazuh, document what I saw. Write-ups live in [detections/](detections/).

| # | Technique | MITRE ATT&CK | Status |
|---|-----------|--------------|--------|
| 1 | *(coming — day 1)* | | ⬜ |
| 2 | *(coming — day 2)* | | ⬜ |
| 3 | *(coming — day 2)* | | ⬜ |

## Case study

One technique, walked through end to end: what I simulated, how it looked in the raw logs, how the rule caught it, and what a false positive for that same rule would look like. Coming in [docs/](docs/) once the detections are done.

## What I learned

Filling this in as I go — the honest version, including what broke and how I figured it out.

## How to reproduce

Coming once the stack is stable, so I don't document steps that turn out to be wrong.
