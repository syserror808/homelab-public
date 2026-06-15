# Proxmox Configuration

---

## Nodes

| Node | Role | Status |
|---|---|---|
| Primary hypervisor | Runs containers and VMs | ✅ Active |
| Proxmox Backup Server | Dedicated backup target | ✅ Active |

---

## Storage

| Type | Content | Purpose |
|---|---|---|
| Directory | ISO, CT templates, local backup | Local storage |
| LVM-Thin | Disk images, Containers | VM/CT storage |
| Proxmox Backup Server | Backup | Offnode backup target |

---

## Containers / VMs

| ID | Type | Name | VLAN | Status |
|---|---|---|---|---|
| 100 | LXC | Pi-hole | Server VLAN | ✅ Running |

### Pi-hole LXC notes

- Static DHCP reservation (MAC-based) outside DHCP pool
- No VLAN tag on container interface (switch port is untagged)
- Pi-hole set to "Permit all origins" for cross-VLAN DNS queries
- Upstream: OPNsense Unbound (recursive, no third-party forwarders)
- lighttpd bound to Server VLAN IP (admin UI restricted)

### Proxmox bridge — VLAN awareness requirement

For containers to pass tagged traffic, the Proxmox host bridge requires:
```
bridge-vlan-aware yes
bridge-vids 2-4094
```
Without this setting, LXC containers on VLAN-aware networks cannot reach
the network regardless of other configuration.

---

## Backup configuration

### Scheduled job

- Frequency: daily at 02:30
- Target: all containers and VMs
- Mode: Snapshot
- Compression: Zstd
- Retention: 7 daily backups (keep-last-7)

### Verification

Backups verified via PBS CLI after first run — snapshot confirmed in datastore
with correct timestamp and file set (catalog, client log, config, root archive).

### Proxmox apt no-subscription fix

Proxmox defaults to the enterprise repo which requires a paid subscription.
For homelab use, replace with the community repo:

```bash
echo "# disabled" > /etc/apt/sources.list.d/pve-enterprise.list
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-community.list
apt update
```

---

## Planned services

| Service | Purpose | Priority |
|---|---|---|
| Grafana + InfluxDB | Network monitoring dashboards | High |
| Wazuh | SIEM / centralized log aggregation | High |
| Vaultwarden | Self-hosted password manager | Medium |
| Nextcloud | Self-hosted file and photo storage | Medium |
| Secondary Pi-hole | DNS redundancy | Medium |
