# Homelab Status

Last updated: 2026-06-15

---

## Current state: Operational

Core infrastructure live. Open NAT achieved for gaming. Management UI
segmentation in progress. All services running.

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

## Recent changes

### Open NAT for gaming (double-NAT environment)

Resolved Strict NAT behind a double-NAT setup (ISP router → OPNsense edge).
Approach:
- ISP router Static NAT → OPNsense WAN (makes OPNsense the true edge)
- OPNsense Outbound NAT: hybrid mode + static port rule (Strict → Moderate)
- OPNsense Destination NAT on gaming ports with correct rule association type
  (Moderate → Open)
- dnsmasq static DHCP reservation for gaming PC MAC to lock the IP permanently

### Management UI segmentation

Two-layer restriction (service-level binding + OPNsense firewall rules) so
admin interfaces (OPNsense, Proxmox, Pi-hole) are only reachable from the
MGMT VLAN:
- OPNsense: listen interface set to MGMT VLAN only
- Proxmox: source-IP filter via pveproxy ALLOW_FROM
- Pi-hole: lighttpd bound to server VLAN IP
- OPNsense block rules on workstation and server VLANs for admin ports

### Suricata IDS confirmed operational

Running in netmap capture mode, alert-only. Inbound internet scan traffic
(common ports like RDP, MSSQL, PostgreSQL) now visible in Suricata logs — this
is expected behavior after making OPNsense the true WAN edge. OPNsense default
deny handles actual enforcement.

---

## In progress

- Replace overly permissive allow-all rule on workstation VLAN with specific
  per-protocol rules to complete inter-VLAN segmentation
- dnsmasq static reservation verification for gaming PC
- Proxmox automated backup scheduling
- Secondary Pi-hole for DNS redundancy
- Mullvad VPN on OPNsense (with policy routing to exclude gaming traffic)

