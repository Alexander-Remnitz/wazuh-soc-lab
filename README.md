# Wazuh SOC Lab

A home SOC (Security Operations Center) lab built on **Wazuh** — an open-source SIEM/XDR — to detect real attacks run against a custom vulnerable target, including a **custom detection rule** written and validated from scratch.

## What this project shows

- Deploying a SIEM (Wazuh all-in-one: indexer, server, dashboard) on an isolated virtual network
- Enrolling agents on an attacker machine (Kali) and a vulnerable target
- Running five realistic attacks and **detecting each one**, mapped to MITRE ATT&CK
- **Writing, validating and deploying a custom detection rule** (detection engineering)
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

Five attacks launched from Kali against the target, each detected in Wazuh:

| # | Attack | Tool | Key rule(s) | Max level | MITRE |
|---|---|---|---|---|---|
| 1 | Port scan | `nmap` | SCA/CIS re-eval (19004/7/8) | 7 | Remote Services |
| 2 | SSH brute force | `hydra` | 5763 (brute force), 2502 | 10 | T1110 Brute Force |
| 3 | Web dir brute force | `gobuster` | 31101, 31151 (correlation) | 10 | Web attack |
| 4 | File integrity tamper | manual | 550 (checksum), 554 (new file) | 7 | Persistence |
| 5 | Vuln endpoint access | `curl` | **100100 (custom rule)** | 10 | T1190 Exploit Public-Facing App |

Evidence for each is in [`screenshots/`](screenshots/). The custom rule is in [`rules/local_rules.xml`](rules/local_rules.xml).

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
