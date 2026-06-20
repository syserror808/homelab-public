# Homelab Status

Last updated: 2026-06-20

---

## Current state: Operational

Core infrastructure live. True VLAN segmentation achieved across all three
VLANs. Automated backups running. Full monitoring stack deployed with all
sources reporting. All services running.

---

## Services

| Service | Role | Status |
|---|---|---|
| OPNsense | Firewall / router / DHCP / DNS relay | ✅ Running |
| Proxmox | Hypervisor / container host | ✅ Running |
| Proxmox Backup Server | Dedicated backup target | ✅ Running |
| Pi-hole (LXC) | Network-wide DNS ad-blocking | ✅ Running |
| InfluxDB v2 (LXC) | Time-series metrics database | ✅ Running |
| Grafana (LXC) | Monitoring dashboards | ✅ Running |
| Telegraf | Metrics collection (multiple hosts) | ✅ Running |
| Unbound | Recursive DNS resolver (DNSSEC) | ✅ Running |
| Suricata | Intrusion detection (alert-only) | ✅ Running |
| Dnsmasq | DHCP + local DNS for all VLANs | ✅ Running |

---

## Completed milestones

### ✅ Full monitoring stack (2026-06-20)
InfluxDB v2 + Telegraf + Grafana deployed, with metrics flowing from the
firewall (native Telegraf plugin), the hypervisor host, the metrics container
itself, and the DNS sinkhole (custom integration). Dashboards built with Flux
queries. See [monitoring.md](monitoring.md) for the full build, including the
custom DNS-sinkhole integration written to work around the lack of a native
agent plugin.

### ✅ Proxmox automated backups
Daily scheduled backups to a dedicated Proxmox Backup Server node, snapshot
mode, Zstd compression, 7-day retention. First backup verified via the backup
server CLI.

### ✅ True VLAN segmentation
All three VLANs enforce proper inter-VLAN isolation via OPNsense firewall
rules. Each VLAN can only reach what it needs — no lateral movement between
workstation, server, and management networks.
- Workstation VLAN: blocked from MGMT and Server, DNS and internet allowed
- Server VLAN: blocked from MGMT and Workstation, DNS upstream and internet allowed
- MGMT VLAN: full access retained for administration
- Verified across all three VLANs

### ✅ Open NAT in double-NAT environment
Achieved Open NAT for gaming behind a double-NAT setup without bridge mode
or DMZ. Three-part solution: ISP Static NAT + outbound static port rule +
destination NAT with correct rule association. Static DHCP reservation anchors
the gaming machine IP permanently.

### ✅ Management UI segmentation
Two-layer restriction (service-level binding + firewall rules) ensures admin
interfaces are only reachable from the MGMT VLAN.

### ✅ DNS filtering + recursive resolution
Full chain: clients → Pi-hole (filtering) → Unbound (recursive, DNSSEC) →
root servers. No third-party DNS forwarders.

### ✅ Suricata IDS
Running in netmap/alert-only mode. Inbound internet scan traffic visible in
logs — expected behavior after making the firewall the true WAN edge.

---

## In progress / next steps

- Expand metrics collection (more granular firewall/host/sinkhole inputs)
- Build out remaining firewall dashboard panels
- Secondary Pi-hole for DNS redundancy
- Additional self-hosted services: SIEM, password manager, file/photo storage
- Single-board-computer cluster for container orchestration (k3s) — learning
  and portfolio project
- VPN with policy routing (exclude gaming traffic from tunnel)
- Suricata tuning after alert data accumulates
