# Proxmox Infrastructure (Public)

Hypervisor cluster and services running on two server nodes.

---

## Nodes

| | Primary | Backup |
|---|---|---|
| Role | Main compute, services | Replication, backup target |
| IP (static) | 10.0.2.10/24 | 10.0.2.11/24 |
| VLAN | 30 (Servers) | 30 (Servers) |
| Switch port | 3 (untagged) | 4 (untagged) |
| Web UI | https://10.0.2.10:8006 | https://10.0.2.11:8007 (PBS) |

Both nodes are mini PCs with modest specs (8GB RAM, small SSD). Sufficient for
light workloads (containers, small VMs).

---

## Proxmox Backup Server (PBS)

Dedicated backup infrastructure on the secondary node.

**Purpose:** Store VM/container backups locally, avoid data loss if primary node fails.

**Storage:** Local filesystem (`/backup`), unencrypted for now. Future: encrypted
when NAS arrives.

**Testing:** Manual backups verified; restore tested.

---

## Pi-hole service

**Type:** LXC container (unprivileged), running Debian

**Function:** Network-wide ad-blocking DNS

**Config:**
- Upstream: firewall's Unbound resolver
- Interface: "Permit all origins" (required for cross-VLAN queries)
- DHCP: all VLANs deliver Pi-hole IP as primary DNS

**Deployment notes:**
- **VLAN-aware bridge required:** without `bridge-vlan-aware yes` in
  `/etc/network/interfaces`, containers got only IPv6 link-local (no IPv4)
- **No VLAN tag on untagged port:** switch port is untagged VLAN 30, so the
  container's network interface must NOT set `tag=30`. Double-tagging broke
  DHCP.
- **Dnsmasq listening on VLAN interfaces:** DHCP was broken until Dnsmasq was
  configured to listen on all VLAN interfaces, not just LAN

**Risks:**
- Single point of failure for DNS across all VLANs
- Mitigation: secondary DNS fallback in DHCP (firewall), secondary Pi-hole
  container planned

---

## Container network (template)

For LXC on untagged access port (e.g., VLAN 30):

```
net0: name=eth0,bridge=vmbr0,hwaddr=...,ip=dhcp,type=veth
```

**Do NOT include:**
- `tag=30` (switch port is untagged)
- IP configuration (use DHCP)

**Must include:**
- `bridge=vmbr0` (VLAN-aware bridge)
- `ip=dhcp` (get IP from OPNsense Dnsmasq)

**Bridge interface config:**

```
auto vmbr0
iface vmbr0 inet static
    address 10.0.2.10/24
    gateway 10.0.2.1
    bridge-ports nic0
    bridge-vlan-aware yes
    bridge-vids 2-4094
    bridge-stp off
    bridge-fd 0
```

The `bridge-vlan-aware yes` is critical — without it, containers can't get
DHCP leases.

---

## Services deployed

- **Pi-hole (LXC):** ad-blocking DNS, running continuously
- **Future:** Docker host for light services (monitoring, dashboards)
- **Future:** Ollama container for local LLM inference (requires GPU passthrough)

---

## Infrastructure improvements

**Completed:**
- Backup infrastructure (PBS) deployed and tested
- Manual backups verified

**To-do:**
- Automatic backup schedule
- VM snapshots before risky changes
- UPS for graceful shutdown during power loss
- Secondary Pi-hole for DNS failover

---

## Troubleshooting reference

**Container won't get IP:** Check bridge is VLAN-aware (`bridge-vlan-aware yes`),
no VLAN tag on the interface (`tag=` should not exist), and Dnsmasq is listening
on all VLAN interfaces (not just LAN).

**Cross-VLAN DNS timeouts:** Pi-hole must have "Permit all origins" enabled in
its DNS settings. Default "local only" restriction blocks queries from other VLANs.

**Repository issues:** Using free no-subscription repo. Update with
`apt update && apt dist-upgrade -y`.

