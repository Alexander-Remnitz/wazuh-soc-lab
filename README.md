# Wazuh SOC Lab

A home SOC (Security Operations Center) lab built on **Wazuh** — an open-source SIEM/XDR — to detect real attacks run against a custom vulnerable target.

> [!NOTE]
> 🚧 **Work in progress.** This README is updated as each phase is completed.

## What this project shows

- Deploying a SIEM (Wazuh all-in-one: indexer, server, dashboard) on an isolated virtual network
- Enrolling agents on an attacker machine (Kali) and a vulnerable target
- Running realistic attacks and **detecting them** in the SIEM
- Writing **custom detection rules** for activity the default ruleset misses

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
| Wazuh | All-in-one deployment (indexer + server + dashboard) |
| Lab network | Isolated libvirt bridge, static IPs, no route to the internet |

## Project status

| Phase | Status |
|---|---|
| 1. Build Wazuh VM (Ubuntu 24.04, dual NIC) | ✅ Done |
| 2. Install Wazuh all-in-one | ⏳ |
| 3. Enroll agents (Kali, target) | ⏳ |
| 4. Attack & detect | ⏳ |
| 5. Custom detection rules | ⏳ |
| 6. Final write-up & screenshots | ⏳ |

## Repository layout

```text
wazuh-soc-lab/
├── README.md
├── docs/
│   └── BUILD_LOG.md      # step-by-step build log, including problems and fixes
├── configs/              # sanitized config snippets (agents, netplan, etc.)
├── rules/                # custom Wazuh detection rules
└── screenshots/          # dashboard evidence of detections
```

## Related projects

- **AI SOC Bot** *(planned)* — a local, private AI (Ollama) that triages the alerts produced by this lab.

## Disclaimer

All attacks in this project are run **only** inside an isolated, self-owned lab network. No credentials, flags, or secrets from the lab are published in this repository.
