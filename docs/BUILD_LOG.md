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

## Phase 2 — Install Wazuh ⏳
