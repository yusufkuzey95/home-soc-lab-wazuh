# Lab Journal

Running notes as I build this thing. Mostly for future me, partly so I remember
what actually went wrong (because something always does).

## Day 1 — getting the SIEM running

Goal for tonight was simple: Wazuh server running in Docker, dashboard in my browser.

What I did:

- Installed Docker Desktop on my Windows 11 machine (winget made this painless).
  Docker Desktop runs a small Linux VM through WSL2 under the hood — that's where
  the containers actually live.
- Cloned the official `wazuh/wazuh-docker` repo, pinned to **v4.14.6**, and used
  the single-node deployment. Single-node means the manager, indexer, and dashboard
  all run on one machine — fine for a lab, but production would cluster the manager
  and indexer so one dead box doesn't blind the SOC.
- Before first launch, set `vm.max_map_count=262144` inside the WSL2 VM. The indexer
  is an OpenSearch database that memory-maps big files, and the kernel default is too
  low. Skip this and the indexer crash-loops forever while the dashboard throws 503s —
  apparently the single most common way a fresh Wazuh-on-Docker install dies.
- Generated the self-signed TLS certs the containers use to talk to each other, then
  `docker compose up -d`. Three containers: **manager** (receives agent logs on 1514,
  runs the detection rules), **indexer** (stores alerts, answers searches), **dashboard**
  (the web UI on 443 that queries the indexer).
- Logged into the dashboard at https://localhost. Browser screamed about the cert
  because it's self-signed — encrypted, but no certificate authority vouches for it.
  Safe to bypass on my own localhost lab, never on a real site.

Things that actually taught me something:

- **"Up" is not "healthy."** `docker ps` saying a container is Up just means the
  process is running. The indexer's own API reporting `"status":"green"` is the real
  health check. If a dashboard is ever empty, check service-level health before
  trusting process-level health.
- **The broken component is often not the one showing the error.** If the indexer
  dies, it's the *dashboard* that looks broken in the browser.
- After a reboot, the containers came back on their own (`restart: always` in the
  compose file) but Docker Desktop itself didn't — the engine needed a manual start.
  Diagnostic order that saved me: browser symptom → is the daemon up → are the
  containers up → is the service actually ready.

Current state: SIEM alive, dashboard shows **0 agents**, which is correct — nothing
is feeding it yet. Tomorrow: enroll a Linux endpoint and get real logs flowing.
