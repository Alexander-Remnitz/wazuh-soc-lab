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

## Phase 4 — Attack & detect ⏳
