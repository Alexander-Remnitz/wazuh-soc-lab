# Suricata (network IDS) integration

Added on the target (`mr-axe`) to give the lab **network-layer** detection,
complementing the host-based Wazuh agent. This is what closes the port-scan gap.

## Why

Wazuh is host-based — it reads logs on the endpoint, so it cannot see a port
scan (no host log is generated). Suricata inspects packets on the wire, so it
*can*. Its alerts are then ingested by the Wazuh agent, unifying both layers in
one SIEM.

## Install & interface

```bash
sudo apt-get install -y suricata jq
sudo suricata-update                 # ET Open ruleset
# point Suricata at the LAB interface (the one attacks arrive on)
sudo sed -i '0,/interface: eth0/s//interface: enp7s0/' /etc/suricata/suricata.yaml
```
- Lab interface on this target: **`enp7s0`** (10.66.66.20). Confirm yours with `ip -br a`.
- `HOME_NET` default already includes `10.0.0.0/8`, covering the lab subnet.

## Custom scan signature

`/var/lib/suricata/rules/local.rules`:
```
alert tcp any any -> $HOME_NET any (msg:"LOCAL Possible TCP port scan - SYN sweep from single source"; flags:S,12; threshold: type both, track by_src, count 20, seconds 10; classtype:attempted-recon; sid:1000001; rev:1;)
```
Include it once in `suricata.yaml` under `rule-files:` (`- local.rules`), then
`sudo suricata -T -c /etc/suricata/suricata.yaml` to validate before restart.

## Feed into Wazuh

`/var/ossec/etc/ossec.conf` on the agent:
```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```
Wazuh has built-in Suricata decoders; scans then appear as Wazuh rule **86601**.

## Verify

```bash
# KALI
sudo nmap -sS -p- 10.66.66.20
# WAZUH — the scan, sourced from Kali:
sudo grep -i "suricata" /var/ossec/logs/alerts/alerts.json | tail -2
```

## Notes
- Default Wazuh Suricata rule fires at level 3; a custom Wazuh rule can escalate
  recon alerts if higher severity is wanted.
- Suricata is memory-hungry — the target VM was raised to 2 GB RAM.
- No secrets in this file; interface names and private-range IPs only.
