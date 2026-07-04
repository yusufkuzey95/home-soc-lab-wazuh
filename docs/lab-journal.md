# Lab journal

Notes as I go. Mostly so I remember what broke and how I fixed it.

## Day 1 - getting the SIEM running

Goal for tonight was just to get the Wazuh server running in Docker and be able to log into the dashboard.

What I did:

- Installed Docker Desktop (winget handled it). Learned that Docker on Windows actually runs everything inside a small Linux VM through WSL2, the containers don't run on Windows directly.
- Cloned the official wazuh-docker repo, checked out v4.14.6, used their single-node setup. Single node means the manager, indexer and dashboard all run on one box. Fine for a lab. A real deployment would cluster the manager and indexer so one dead server doesn't blind the whole SOC.
- Had to set vm.max_map_count=262144 inside the WSL VM before starting anything. The indexer is OpenSearch under the hood and it memory-maps big files, and the kernel default is too low for that. If you skip this the indexer just crash-loops forever and the dashboard sits there throwing 503s. Apparently this is the number one way fresh Wazuh installs die.
- Generated the self-signed certs the containers use to talk to each other, then docker compose up -d. Three containers: the manager (receives logs from agents on tcp 1514, runs the detection rules), the indexer (stores the alerts, answers searches) and the dashboard (web UI on 443).
- Logged in at https://localhost. The browser freaks out about the certificate because it's self-signed. The connection is still encrypted, there's just no certificate authority vouching for who the server is. That's fine on my own localhost, it would not be fine on a real website.

Stuff that actually taught me something:

- docker ps saying "Up" only means the process is running, not that the service works. The indexer has its own health API that returns green/yellow/red, and that's the check that matters. Container up + cluster red means stop looking at Docker and go read the service logs.
- When the indexer died in my testing, the thing that looked broken in the browser was the dashboard. The component showing the error is not always the component that broke.
- After a reboot nothing worked, and it turned out Docker Desktop doesn't start with Windows by default. The containers themselves came back on their own once the engine was up (restart: always in the compose file). Debug order that saved me: browser symptom -> is the docker daemon running -> are the containers up -> is the service actually healthy.

Ended the day with the dashboard showing 0 agents, which is correct, nothing is enrolled yet. Endpoint and agent are tomorrow's job.

## Day 2 - endpoint enrolled, logs flowing

Morning surprise: dashboard wouldn't load. Turned out Docker Desktop doesn't start with Windows, so the whole SIEM was just off. Started it, containers came back on their own. Good reminder that "the SIEM is down" can mean something as dumb as the host never booted the engine.

Then the endpoint. I went with Ubuntu 24.04 on WSL2 instead of a full VM, it's a real Ubuntu userspace with systemd and it was up in ten minutes instead of an hour of ISO downloads.

First thing before installing anything: checked whether the endpoint can even reach the manager. Tested ports 1514/1515 from inside Ubuntu against both localhost and the WSL gateway IP, both worked. Used localhost in the agent config anyway, because the gateway IP is handed out dynamically and changes across reboots. An agent pinned to an IP that moves is an agent that silently dies next week. Real deployments use a DNS name for the same reason.

Install itself:

- added Wazuh's apt repo with their GPG key (so apt can verify the packages actually come from Wazuh)
- installed rsyslog (the minimal WSL image doesn't even write /var/log/auth.log without it), openssh-server (the thing I'll be attacking), and wazuh-agent 4.14.6, same version as the manager
- WAZUH_MANAGER=localhost at install time bakes the manager address into the agent config

Then I got to watch the enrollment happen live in /var/ossec/logs/ossec.log:

```
Requesting a key from server: localhost
Valid key received
```

That's the one-time registration on port 1515. The key is what authenticates everything the agent ships afterward on 1514, so nobody can spoof my endpoint and feed garbage into the SIEM.

Verified the connection from both ends, because one side of a link lying to you is a thing:

- agent side: status='connected' in the agent state file
- manager side: agent_control -l shows ID 001, ubuntu-endpoint, Active

And the part I didn't expect: 282 alerts in the indexer before I attacked anything. The agent ran a CIS benchmark audit against the box (Wazuh calls it SCA) and scored my fresh Ubuntu at 51% hardened. So the agent does two different jobs: it audits the machine's state (how easy would this box be to break into) and it watches events in real time (is someone actually trying). Home inspector and burglar alarm. Most of a SOC queue is the inspector, not the alarm.

Next up: the first real detection, SSH brute force.
