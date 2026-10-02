# Wazuh SOC Lab — Build Log

Chronological log of every step, including the problems hit and how they were fixed.

---

## Phase 1 — Build the Wazuh VM (2026-09-30) ✅

### 1.1 Check the host

```bash
# OMARCHY
virsh -c qemu:///system net-list --all
virsh -c qemu:///system list --all
df -h /var/lib/libvirt/images
free -h
virsh -c qemu:///system net-dumpxml ctf-isolated
```

Findings:
- `ctf-isolated` is a pure layer-2 bridge (`virbr-ctf`): no `<forward>`, no `<ip>`, no DHCP → every VM needs a **static IP**.
- Existing lab IPs: Kali `10.66.66.10`, target `10.66.66.20` → Wazuh gets `10.66.66.30`.
- Wazuh's installer needs the internet, so the VM gets a **second NIC on the `default` NAT network**.

> [!TIP] Gotcha — `qemu:///system`
> On Arch-based hosts, plain `virsh` as a normal user may connect to `qemu:///session` and show **no** VMs or networks. Always use `-c qemu:///system` or `sudo virsh`.

### 1.2 Download and verify Ubuntu Server 24.04

```bash
# OMARCHY
cd ~/Downloads
curl -LO https://releases.ubuntu.com/24.04/ubuntu-24.04.5-live-server-amd64.iso
curl -LO https://releases.ubuntu.com/24.04/SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing   # → OK
```

> [!TIP] Gotcha — no `wget`
> Omarchy doesn't ship `wget`; `curl -LO` does the same job.

### 1.3 Create the VM

```bash
# OMARCHY
sudo mv ~/Downloads/ubuntu-24.04.5-live-server-amd64.iso /var/lib/libvirt/images/
sudo virsh net-start default

sudo virt-install \
  --connect qemu:///system \
  --name wazuh-soc \
  --memory 8192 \
  --vcpus 4 \
  --cpu host-passthrough \
  --disk path=/var/lib/libvirt/images/wazuh-soc.qcow2,size=50,format=qcow2 \
  --location /var/lib/libvirt/images/ubuntu-24.04.5-live-server-amd64.iso,kernel=casper/vmlinuz,initrd=casper/initrd \
  --extra-args 'console=ttyS0,115200n8' \
  --os-variant ubuntu24.04 \
  --network network=default,model=virtio \
  --network network=ctf-isolated,model=virtio \
  --graphics none
```

Why these choices:
- **4 vCPU / 8 GB / 50 GB** = Wazuh's recommended all-in-one spec for 1–25 agents.
- **`--location` + `console=ttyS0`** runs the Ubuntu installer as text inside the terminal (headless, no GUI).
- **NIC order:** first NIC (`enp1s0`) → `default` NAT, second NIC (`enp2s0`) → `ctf-isolated`.

Installer choices: Ubuntu Server (not minimized) · hostname `wazuh` · user `labadmin` · OpenSSH server installed · no snaps · Ubuntu Pro skipped.

### 1.4 Post-install fixes

Verification showed two problems:

| Problem | Cause | Fix |
|---|---|---|
| Root filesystem only **24 GB** of 50 GB | Ubuntu's guided LVM layout only allocates ~half the disk | Grow the LV online (below) |
| `enp2s0` **DOWN, no IP** | Static IP from the installer didn't persist | Dedicated netplan file (below) |

**Grow the disk (safe while running):**
```bash
# WAZUH
sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
df -h /    # → 48G
```

**Static lab IP on `enp2s0`:**
```bash
# WAZUH
sudo tee /etc/netplan/60-ctf-isolated.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    enp2s0:
      dhcp4: false
      addresses: [10.66.66.30/24]
EOF
sudo chmod 600 /etc/netplan/60-ctf-isolated.yaml
sudo netplan apply
```

> [!WARNING] Gotcha — `netplan apply` timing
> Checking `ip -br a` immediately after `netplan apply` showed **no IPv4 on either NIC**. It was only the networking restart in progress — a few seconds later both addresses were back.

> [!TIP] Gotcha — running commands on the wrong machine
> Check the prompt before running anything: Omarchy is `~ ❯`, the Wazuh VM is `labadmin@wazuh:~$`.

### 1.5 Final verification

| Check | Result |
|---|---|
| Hostname | `wazuh` ✅ |
| `enp1s0` | `192.168.122.123/24` (DHCP, internet) ✅ |
| `enp2s0` | `10.66.66.30/24` (static, lab) ✅ |
| Default route | via `enp1s0` only — lab NIC has no internet route ✅ |
| Disk | 48 GB ✅ |
| RAM | 7.8 GB ✅ |
| `net.ipv4.ip_forward` | `0` — VM does not route between networks ✅ |
| Survives reboot | ✅ |

### 1.6 Update and snapshot

```bash
# WAZUH
sudo apt update && sudo apt full-upgrade -y
sudo reboot

# OMARCHY
sudo virsh snapshot-create-as wazuh-soc base-ubuntu-clean "Ubuntu 24.04 updated, both NICs configured, before Wazuh"
```

Snapshot `base-ubuntu-clean` = clean restore point before Wazuh is installed.

---

## Phase 2 — Install Wazuh (2026-09-30) ✅

### 2.1 Run the all-in-one installer

```bash
# WAZUH
tmux new -s wazuh
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

- Installs **Wazuh 4.14.8**: indexer → server (manager + Filebeat) → dashboard, all on one host.
- Took ~15 minutes. The indexer step is the slowest and prints nothing for several minutes — that's normal.
- Run inside **`tmux`** so an SSH drop can't kill the install (`tmux attach -t wazuh` to reconnect).
- The installer creates `wazuh-install-files.tar`, which contains **all generated passwords** → excluded via `.gitignore`, never committed.

### 2.2 Post-install hardening

```bash
# WAZUH
sudo chmod 600 ~/wazuh-install-files.tar                            # restrict the password archive
sudo sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list     # stop accidental Wazuh upgrades
sudo apt update
```

Disabling the Wazuh repo is recommended by Wazuh: an unplanned upgrade of one component can break the stack.

### 2.3 Dashboard login

Dashboard: `https://192.168.122.123` (self-signed certificate → browser warning is expected).

> [!TIP] Gotcha — username is case-sensitive
> `Admin` fails with "Invalid username or password"; it must be `admin`.

> [!TIP] Gotcha — copying passwords from tmux
> Generated passwords contain `*`, `+`, `.`, `?`. Copying from a tmux pane can add spaces or line breaks. Reliable source:
> ```bash
> sudo tar -O -xf ~/wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt | grep -A1 "indexer_username: 'admin'"
> ```

### 2.4 Rotate the admin password

```bash
# WAZUH
sudo bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -u admin
sudo systemctl restart filebeat wazuh-manager wazuh-dashboard
```

- Omitting `-p` makes the tool **generate a random password**, so it never lands in shell history.
- On all-in-one, the tool also updates Filebeat's keystore automatically.

> [!WARNING] Lesson learned — credential hygiene
> The password was accidentally exposed twice (visible in a screenshot, then in pasted terminal output) and had to be rotated each time. Rule going forward: **scan every screenshot/paste for passwords and redact before sharing.**

### 2.5 Verification

```bash
# WAZUH
sudo systemctl is-active wazuh-indexer wazuh-manager filebeat wazuh-dashboard   # → active ×4
```

| Check | Result |
|---|---|
| All 4 services | active ✅ |
| Dashboard login with rotated password | ✅ |
| Baseline alerts (no agents yet) | ~320 medium/low — the manager monitors **its own host**, so installs, package changes and logins already generate events |

### 2.6 Snapshot

```bash
# OMARCHY
sudo virsh snapshot-create-as wazuh-soc wazuh-installed "Wazuh 4.14.8 all-in-one installed, admin password rotated"
```

---

## Phase 3 — Enroll agents (2026-09-30) ✅

### 3.1 RAM budget (16 GB host)

With everything running, RAM is the main constraint:

| VM | RAM |
|---|---|
| Wazuh server | 8 GB |
| Target (mr-axe) | 1 GB |
| Kali | **2.5 GB** (reduced from 4 GB) |
| Omarchy host + browser | remainder |

```bash
# OMARCHY — lower Kali's RAM (VM must be off; persists across boots)
sudo virsh setmem kali-redteam 2560M --config
sudo virsh setmaxmem kali-redteam 2560M --config
```

> [!TIP]
> During lab sessions, close heavy browser tabs and keep only the Wazuh dashboard open.

### 3.2 Install the agent (same on both endpoints)

```bash
# DEBIAN (mr-axe) / KALI — via each VM's internet-facing NIC
sudo apt-get update
sudo apt-get install -y gnupg apt-transport-https curl
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt-get update

sudo WAZUH_MANAGER="10.66.66.30" WAZUH_AGENT_NAME="<agent-name>" apt-get install -y wazuh-agent=4.14.8-1
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent

sudo sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list   # prevent accidental agent upgrades
sudo apt-get update
```

Design decisions:
- **`WAZUH_MANAGER="10.66.66.30"`** — agents report over the isolated lab network, not the NAT network.
- **Pinned to `4.14.8-1`** — Wazuh only guarantees compatibility when the agent is **not newer** than the manager.
- **Repo disabled afterwards** — same reason as on the server.

### 3.3 Verification

**Enrolled agents:**
```bash
# WAZUH
sudo /var/ossec/bin/agent_control -l
```
```text
ID: 000, Name: wazuh (server), IP: 127.0.0.1, Active/Local
ID: 001, Name: mr-axe, IP: any, Active
ID: 002, Name: kali-redteam, IP: any, Active
```
(`IP: any` = the agent may connect from any address; normal for this enrollment method.)

**Agent traffic really uses the lab network:**
```bash
# WAZUH
sudo ss -tn state established '( sport = :1514 )'
```
```text
10.66.66.30:1514   ←   10.66.66.20   (mr-axe)
10.66.66.30:1514   ←   10.66.66.10   (kali-redteam)
```
Both agents connect over `ctf-isolated` ✅ — no agent traffic crosses the NAT network.

### 3.4 Snapshots

```bash
# OMARCHY
sudo virsh snapshot-create-as mr.axe pre-wazuh-agent "CTF box before installing Wazuh agent"
sudo virsh snapshot-create-as mr.axe wazuh-agent-installed "CTF box with Wazuh agent 4.14.8 enrolled"
```

> [!WARNING] Gotcha — Kali can't take internal snapshots
> `error: Operation not supported: internal snapshots of a VM with pflash based firmware require QCOW2 nvram format`
>
> The Kali VM boots with **UEFI** firmware whose NVRAM is stored as a raw file, and libvirt only supports internal snapshots of UEFI VMs when NVRAM is qcow2. Decision: **no snapshots for Kali** — it's the attacker machine, holds no lab state worth rolling back, and the agent can simply be removed with `apt purge wazuh-agent`. If needed later: cold-copy its disk while the VM is shut off.

> [!NOTE] Impact on the CTF box
> The target now runs a Wazuh agent (as real servers run EDR/SIEM agents). The CTF's own build docs and provisioning script should record this change.

---

## Phase 4 — Attack & detect (2026-09-30 to 2026-10-02) — 6/6 verified

### 4.1 Enter "play" state

Isolate the lab before attacking: both internet-facing NICs down, Wazuh's dashboard NIC stays up.

```bash
# OMARCHY — bring target's management NIC down (MAC of its 'default' NIC)
sudo virsh domif-setlink mr.axe 52:54:00:9e:e6:80 down
```
```bash
# KALI — disable internet NIC, verify isolation
sudo ip link set eth0 down
ping -c2 8.8.8.8      # fails  → no internet
ping -c2 10.66.66.20  # replies → reaches target
ping -c2 10.66.66.30  # replies → reaches Wazuh
```

Agents keep reporting during play because they talk to Wazuh over `ctf-isolated` (10.66.66.30), which stays up.

> [!NOTE] Trade-off
> With the target's mgmt NIC down, there's no SSH to mr-axe during play. That's fine for attacking (everything is driven from Kali), but any config change on the target means briefly bringing the mgmt NIC back up.

### 4.2 Attack 1 — Port scan → gap found, then closed with a network IDS ✅

```bash
# KALI
sudo nmap -sS -p- 10.66.66.20      # open: 22/ssh, 80/http
```

**Initial result: Wazuh did NOT detect the scan.** Wazuh is **host-based** — it
analyses logs on the endpoint and has no packet visibility, so a raw SYN scan
produces no Wazuh alert. The activity on the agent during the nmap window was
routine **CIS/SCA configuration re-checks** (19004/19007/19008), not scan
detection.

**Fix: added Suricata (network IDS) feeding Wazuh.** See Phase 7 below. After
integration the same nmap scan is detected end-to-end: Suricata flags the SYN
sweep, its `eve.json` is ingested by the Wazuh agent, and Wazuh fires rule
**86601** (`Suricata: Alert - … port scan`) with `src_ip 10.66.66.10` (Kali).

> [!TIP] The lesson that matters
> A host-based SIEM structurally cannot see network recon. Recognising that and
> closing it with the right tool (a network IDS) is the skill — not pretending
> the host-based SIEM caught something it can't.

### 4.3 Attack 2 — SSH brute force → detected ✅

```bash
# KALI
hydra -l labadmin -P /usr/share/wordlists/rockyou.txt -t 4 -f ssh://10.66.66.20
```
Wazuh escalated correctly:
- **5760** (level 5) — individual `sshd: authentication failed`
- **5763** (level 10) — `sshd: brute force trying to get access` — correlation rule
- **2502** (level 10) — `syslog: user missed the password more than one time`

The 5763 rule logic (from its definition): **8 failures / 120 s / same source IP** → escalate. MITRE T1110 (Brute Force), mapped to PCI DSS, HIPAA, NIST.

> [!TIP] Reading the source IP matters
> An early 5760 alert turned out to be sourced from the **host** (192.168.122.1) during setup, not Kali. The SIEM logs everything — always check `data.srcip` before attributing an alert.

### 4.4 Attack 3 — Web attack → detected ✅ (after real troubleshooting)

```bash
# KALI
gobuster dir -u http://10.66.66.20 -w /usr/share/wordlists/dirb/common.txt -t 20
```
Result: **2,031 alerts** —
- **31101** (level 5) — `Web server 400 error code` (every 404 from the wordlist)
- **31151** (level 10) — `Multiple web server 400 error codes from same source ip` (correlation)

> [!WARNING] Lesson learned — Wazuh only watches logs you tell it to
> The web attack produced **zero** alerts at first. Root cause chain:
> 1. The Wazuh Linux agent does **not** monitor Apache logs by default (SSH worked only because it's read via journald).
> 2. First fix pointed FIM/logcollector at `/var/log/apache2/access.log` — still nothing.
> 3. `access.log` was **empty**: the CTF's vhost (`project-zero.conf`) logs to a **custom path**, `CustomLog .../project-zero-access.log`.
> 4. Repeated `sed` edits left **duplicate / malformed `<localfile>` blocks** (some missing `<location>`), which crashed the agent.
> 5. Final fix: rewrote `ossec.conf` cleanly with **one** correct block pointing at `project-zero-access.log`, then re-tested.
>
> Takeaway: confirm the *actual* log path (`apachectl -S`), verify the file is being written (`curl` + `tail`), and validate config changes instead of blind-appending.

Correct FIM/log block added to the target's `ossec.conf`:
```xml
<localfile>
  <log_format>apache</log_format>
  <location>/var/log/apache2/project-zero-access.log</location>
</localfile>
```

### 4.5 Attack 4 — File Integrity Monitoring → detected ✅

```bash
# DEBIAN (mr-axe) — modify a monitored file + drop a fake binary
sudo bash -c 'echo "10.66.66.99 realtime-test-c2" >> /etc/hosts'
sudo touch /usr/bin/realtime-backdoor
```
Wazuh:
- **554** — `File added to the system` (`/usr/bin/realtime-backdoor`)
- **550** — `Integrity checksum changed` (`/etc/hosts`)

> [!WARNING] Lesson learned — default FIM is periodic, not realtime
> Default syscheck scans every **12 h** (`frequency 43200`), and its scan of the listed dirs finishes in ~1 s, so edits made between scans produced no alerts. Manual `pkill -USR1` did not force a rescan reliably. Fix: enable realtime on the watched dirs:
> ```xml
> <directories realtime="yes">/etc,/usr/bin,/usr/sbin</directories>
> ```
> After restart, new changes were caught **instantly**.

> [!TIP] Gotcha — agent went Disconnected
> Running `wazuh-control restart` on the target right as the mgmt NIC was dropped left the agent `Disconnected` and its scan killed mid-run. Restart the agent, confirm `agent_control -i 001` shows **Active**, and only then drop the NIC.

### 4.6 Attack 5 — Custom detection rule → detected ✅

The first four attacks proved the **default** ruleset detects generic activity.
Attack 5 shows **detection engineering**: writing a rule for something the
defaults don't know about — a sensitive endpoint specific to *this* environment.

The target hosts a deliberately vulnerable PHP page, `diag.php` (command
injection). Wazuh's default web rules only flag generic 400/404 patterns; they
have no concept that `diag.php` is a crown-jewel endpoint here. So we add that
knowledge as a custom rule.

**Rules live on the MANAGER**, not the agent — `/var/ossec/etc/rules/local_rules.xml`.

```xml
<group name="web,attack,local,">
  <rule id="100100" level="10">
    <if_group>web</if_group>
    <url>diag.php</url>
    <description>Access attempt to known-vulnerable endpoint (diag.php) on mr-axe</description>
    <mitre>
      <id>T1190</id>
    </mitre>
    <group>attack,web_attack,</group>
  </rule>
</group>
```
- **id 100100** — user-rule range (must be ≥ 100000).
- **`if_group web`** — only evaluated on already-decoded web events (efficient).
- **level 10**, **MITRE T1190** (Exploit Public-Facing Application, Initial Access).

**Validate before loading** — `wazuh-logtest` tests a rule safely without a restart:
```bash
# WAZUH
echo '10.66.66.10 - - [30/Sep/2026:10:00:00 +0000] "GET /login/diag.php HTTP/1.1" 200 100 "-" "curl/8.0"' \
  | sudo /var/ossec/bin/wazuh-logtest
```
Output confirmed: decoded `url: /login/diag.php`, then **Phase 3 matched rule 100100 / level 10**, MITRE T1190 → "Alert to be generated."

**Load and trigger:**
```bash
# WAZUH
sudo systemctl restart wazuh-manager

# KALI
curl -s -o /dev/null "http://10.66.66.20/login/diag.php"
curl -s -o /dev/null "http://10.66.66.20/login/diag.php?cmd=id"
```
Dashboard (`rule.id: 100100`): **4 live alerts**, level 10, from mr-axe — the custom rule firing on real requests from the Kali attacker.

> [!TIP] Workflow that works
> write rule → `wazuh-logtest` → restart manager → trigger → confirm in dashboard.
> Always logtest first: it catches XML errors without restarting the manager.

### 4.7 Detection summary

| Attack (from Kali) | Detected? | Key rule(s) | Max level |
|---|---|---|---|
| SSH brute force (hydra) | ✅ | 5763 brute force, 2502 | 10 |
| Web dir brute force (gobuster) | ✅ | 31101 → 31151 | 10 |
| Access to `diag.php` | ✅ (custom) | **100100** | 10 |
| File tamper + SUID bit set | ✅ (realtime FIM) | 554, 550 (perm → SUID) | 7 |
| Privileged command (`sudo`) | ✅ | 5402 (full command logged) | 3 |
| Port scan (nmap) | ✅ (via Suricata) | 86601 + custom sig 1000001 | 3 |

Six attack techniques detected across **two layers**: host-based (Wazuh agent)
and network-based (Suricata IDS → Wazuh). The port scan was initially a gap in
the host-based design, then closed by adding Suricata (Phase 7).

---

## Phase 5 — Deliverables committed ✅

- `rules/local_rules.xml` — the custom rule, committed to the repo.
- `configs/agent-ossec-snippets.conf` — the Apache-log + realtime-FIM additions
  made on the target agent (sanitized).
- `screenshots/` — detection evidence for all six detections (01–06); privileged-command rule 5402 was re-verified live on 2026-10-02.

---

## Phase 6 — Finalize ✅

- README polished; project status table completed.
- Repository made **public** (contains no secrets — verified against `.gitignore`).
- Obsidian notes written to the vault, matching the CTF project's note style.
- CTF project docs updated to record that the target now runs a Wazuh agent.

### What this project demonstrates
- Standing up a full SIEM (Wazuh all-in-one) on an isolated network.
- Enrolling agents and proving traffic stays on the isolated segment.
- Detecting six distinct attack types across host + network layers, mapped to MITRE ATT&CK.
- **Writing and validating custom detection rules** — a Wazuh rule *and* a Suricata signature (detection engineering).
- **Identifying an architectural blind spot and closing it** (host-based SIEM → added network IDS).
- Real troubleshooting: wrong Apache log path, periodic-vs-realtime FIM,
  credential hygiene, agent connectivity, Suricata capture interface — documented honestly.

---

## Phase 7 — Network IDS: closing the port-scan gap (2026-10-01) ✅

The host-based SIEM couldn't see network recon (4.2). Added **Suricata** on the
target to give the lab network-layer visibility, feeding Wazuh.

### 7.1 Install & point at the lab interface

```bash
# DEBIAN (mr-axe) — target has two NICs; the lab one is enp7s0 (10.66.66.20)
sudo apt-get install -y suricata jq
sudo suricata-update                                   # pull ET Open rules
sudo sed -i '0,/interface: eth0/s//interface: enp7s0/' /etc/suricata/suricata.yaml
```
`HOME_NET` already covered `10.0.0.0/8`, so no change needed there.

### 7.2 Custom scan signature

Default rules don't reliably flag a quiet SYN scan, so a deterministic signature
was added — fires when one source sends 20+ SYNs in 10 s:

```
alert tcp any any -> $HOME_NET any (msg:"LOCAL Possible TCP port scan - SYN sweep from single source"; \
  flags:S,12; threshold: type both, track by_src, count 20, seconds 10; \
  classtype:attempted-recon; sid:1000001; rev:1;)
```
Validated with `suricata -T` (dedupe the `local.rules` include first, or you get
a "duplicate signature" error).

### 7.3 Feed Suricata into Wazuh

```xml
<!-- added to /var/ossec/etc/ossec.conf on the target agent -->
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```
Wazuh ships built-in Suricata decoders, so `eve.json` alerts are parsed automatically.

### 7.4 Verified end-to-end

```bash
# KALI
sudo nmap -sS -p- 10.66.66.20
```
Result — the scan surfaced as a Wazuh alert:
```json
"rule": {"description":"Suricata: Alert - LOCAL Possible TCP port scan ...","id":"86601","groups":["ids","suricata"]}
"data": {"in_iface":"enp7s0","src_ip":"10.66.66.10","dest_ip":"10.66.66.20","alert":{"signature_id":"1000001"}}
```
Source confirmed as Kali (`10.66.66.10`). The lab now detects at **two layers**.

> [!NOTE] Trade-off recorded honestly
> The default Wazuh Suricata rule fires at level 3. Fine for proving detection;
> a custom Wazuh rule could escalate recon alerts if desired. Also bumped the
> target VM to 2 GB RAM — Suricata with a full rule set is memory-hungry.
