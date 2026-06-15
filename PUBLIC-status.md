# Homelab Status

Last updated: 2026-06-15

---

## Current state: Operational

Core infrastructure live. True VLAN segmentation achieved across all three
VLANs. Open NAT confirmed. All services running.

---

## Services

| Service | Role | Status |
|---|---|---|
| OPNsense | Firewall / router / DHCP / DNS relay | ✅ Running |
| Proxmox | Hypervisor / container host | ✅ Running |
| Pi-hole (LXC) | Network-wide DNS ad-blocking | ✅ Running |
| Unbound | Recursive DNS resolver (DNSSEC) | ✅ Running |
| Suricata | Intrusion detection (alert-only) | ✅ Running |
| Dnsmasq | DHCP + local DNS for all VLANs | ✅ Running |

---

## Completed milestones

### ✅ True VLAN segmentation (2026-06-15)
All three VLANs now enforce proper inter-VLAN isolation via OPNsense firewall
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

- Proxmox backup automation (top priority — no automated backups yet)
- Secondary Pi-hole for DNS redundancy
- Mullvad VPN with policy routing (exclude gaming traffic from tunnel)
- Proxmox service expansion: Grafana, Wazuh, Vaultwarden, Nextcloud
- Suricata tuning after alert data accumulates
