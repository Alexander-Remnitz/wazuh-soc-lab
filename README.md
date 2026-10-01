# Wazuh SOC Lab

A home SOC (Security Operations Center) lab built on **Wazuh** — an open-source SIEM/XDR — to detect real attacks run against a custom vulnerable target, including a **custom detection rule** written and validated from scratch.

## What this project shows

- Deploying a SIEM (Wazuh all-in-one: indexer, server, dashboard) on an isolated virtual network
- Enrolling agents on an attacker machine (Kali) and a vulnerable target
- Running realistic attacks and **detecting them across two layers** — host-based (Wazuh) and network-based (Suricata IDS) — mapped to MITRE ATT&CK
- **Writing, validating and deploying custom detection content** — a Wazuh rule and a Suricata signature (detection engineering)
- **Finding an architectural blind spot and closing it** (host-based SIEM couldn't see port scans → added a network IDS)
- Honest troubleshooting notes — the problems hit and how they were fixed

## Architecture

```text
Omarchy host (QEMU/KVM + libvirt)
│
├── default (NAT, 192.168.122.0/24) ── internet for installs, dashboard access from host
│
└── ctf-isolated (10.66.66.0/24, no DHCP, no internet)
    ├── Kali attacker ........ 10.66.66.10   (Wazuh agent)
    ├── Vulnerable target .... 10.66.66.20   (Wazuh agent)
    └── Wazuh server ......... 10.66.66.30   (+ 192.168.122.x mgmt NIC)
```

| Component | Details |
|---|---|
| Hypervisor | QEMU/KVM + libvirt on Omarchy (Arch Linux) |
| Wazuh VM | Ubuntu Server 24.04 LTS · 4 vCPU · 8 GB RAM · 50 GB disk |
| Wazuh | All-in-one deployment (indexer + server + dashboard), v4.14.8 |
| Lab network | Isolated libvirt bridge, static IPs, no route to the internet |

## Detections

Attacks launched from Kali against the target, and what Wazuh saw:

| Attack | Tool | Detected? | Key rule(s) | MITRE |
|---|---|---|---|---|
| SSH brute force | `hydra` | ✅ Yes | 5763 (correlation), 2502 | T1110 Brute Force |
| Web dir brute force | `gobuster` | ✅ Yes | 31101 → 31151 (correlation) | Web attack |
| Access to vuln endpoint | `curl` | ✅ Yes (**custom rule**) | 100100 | T1190 Exploit Public-Facing App |
| File tamper / SUID set | manual | ✅ Yes (realtime FIM) | 554 (new file), 550 (perms → SUID) | T1565.001 |
| Privileged command | `sudo` | ✅ Yes | 5402 (sudo to root, full command logged) | T1548.003 |
| Port scan | `nmap` | ✅ Yes (**via Suricata IDS**) | 86601 (Suricata) + custom sig 1000001 | T1046 Network Service Discovery |

**The port scan — a gap I found, then closed.** Wazuh is **host-based** — it reads
logs on the endpoint and has no packet visibility, so on its own it does **not**
see a port scan (during nmap, the only host activity was routine CIS/SCA
re-checks — not scan detection). Rather than leave that gap, I added **Suricata**
(a network IDS) on the target, watching the lab interface. Suricata inspects the
packets, a custom signature flags the SYN sweep, and its `eve.json` alerts are
ingested by the Wazuh agent — so the scan now surfaces as Wazuh rule **86601**,
with Kali (`10.66.66.10`) as the confirmed source. The lab now does **layered
detection: host-based (Wazuh) + network-based (Suricata)**. See
[`configs/suricata-notes.md`](configs/suricata-notes.md).

Evidence for each detection is in [`screenshots/`](screenshots/). The custom rule
is in [`rules/local_rules.xml`](rules/local_rules.xml).

## Custom detection rule

The default ruleset flags generic web errors but doesn't know that this
environment's `diag.php` is a sensitive, command-injectable endpoint. So a
custom rule (id `100100`, level 10, MITRE T1190) was added on the manager,
validated with `wazuh-logtest`, and confirmed firing live from the attacker.
This is the SIEM-tuning a SOC does for its own crown-jewel assets — see
[the build log](docs/BUILD_LOG.md#46-attack-5--custom-detection-rule--detected-) for the full workflow.

## Project status

| Phase | Status |
|---|---|
| 1. Build Wazuh VM (Ubuntu 24.04, dual NIC) | ✅ Done |
| 2. Install Wazuh all-in-one (4.14.8) | ✅ Done |
| 3. Enroll agents (Kali, target) | ✅ Done |
| 4. Attack & detect (5/5) | ✅ Done |
| 5. Custom detection rule | ✅ Done |
| 6. Write-up, screenshots, published | ✅ Done |

## Repository layout

```text
wazuh-soc-lab/
├── README.md
├── docs/
│   └── BUILD_LOG.md       # step-by-step build log, including problems and fixes
├── configs/
│   └── agent-ossec-snippets.conf   # sanitized agent config additions
├── rules/
│   └── local_rules.xml    # custom Wazuh detection rule (id 100100)
└── screenshots/           # dashboard evidence, one folder per attack
    ├── 01-portscan/
    ├── 02-ssh-bruteforce/
    ├── 03-web-attack/
    ├── 04-fim/
    └── 05-custom-rule/
```

## Key lessons (documented in the build log)

- Wazuh only monitors the logs you tell it to — Apache logging is opt-in, and the vhost used a **custom log path**.
- Default File Integrity Monitoring is **periodic (12 h)**, not realtime — enabling `realtime="yes"` made changes fire instantly.
- Detection-engineering workflow that works: **write rule → `wazuh-logtest` → restart manager → trigger → confirm**.
- Credential hygiene: rotate any secret that lands in a screenshot or paste.

## Related projects

- **[AI SOC Bot](https://github.com/Alexander-Remnitz/ai-soc-bot)** — a local, private AI (Ollama) that triages the alerts produced by this lab. This lab *detects*; the bot *automates the triage*.

## Disclaimer

All attacks in this project are run **only** inside an isolated, self-owned lab network. No credentials, flags, or secrets from the lab are published in this repository.
