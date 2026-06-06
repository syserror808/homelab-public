# OPNsense Configuration (Public)

Firewall and routing configuration for the homelab. All IP addresses and specific
hardware details have been sanitized.

---

## Interfaces

| Role | Device | Config | Purpose |
|------|--------|--------|---------|
| WAN | Primary NIC | DHCP (private subnet) | Internet uplink |
| MGMT (OPT1) | VLAN 10 trunk | Static 10.0.0.1/24 | Management VLAN |
| Primary (OPT2) | VLAN 20 trunk | Static 10.0.1.1/24 | Primary device VLAN |
| Servers (OPT3) | VLAN 30 trunk | Static 10.0.2.1/24 | Hypervisor VLAN |

All interfaces use Intel GbE NICs (resolved Realtek watchdog timeout issues on
previous hardware).

---

## DHCP Service (Dnsmasq)

**Critical configuration:** The DHCP service must listen on **all** VLAN interfaces,
not just the primary LAN. This was the root cause of a multi-day outage.

```
Listening interfaces: Management, Primary, Servers
(DO NOT include WAN)
```

### DHCP pools

| VLAN | Range |
|------|-------|
| Management | 10.0.0.100–10.0.0.200 |
| Primary | 10.0.1.100–10.0.1.200 |
| Servers | 10.0.2.100–10.0.2.200 |

### Static reservations

Reserved addresses (outside pools):
- Ad-blocking DNS service: 10.0.2.50
- Primary hypervisor: 10.0.2.10
- Backup hypervisor: 10.0.2.11

### DNS via DHCP option 6

All VLANs have DHCP option 6 set to deliver:
1. Primary: ad-blocking DNS service (10.0.2.50)
2. Secondary: firewall recursive resolver (10.0.2.1) — fallback if primary is down

Devices pick up the new DNS on lease renewal.

---

## DNS Resolver (Unbound)

**Full recursive resolution** — no upstream forwarding. Unbound queries root
servers directly for all lookups.

- Listening on port 53, all interfaces
- DNSSEC enabled and enforced
- Caching enabled

### DNS flow

```
clients
  ↓ (query for domain)
ad-blocking DNS service (filters ads)
  ↓ (ad-free query)
OPNsense Unbound (recursive resolution)
  ↓
root servers
```

No DNS loops: the ad-blocking service does not forward back to itself.

---

## Quality of Service (Traffic Shaper)

Configured to maintain gaming/streaming performance on the primary device
despite concurrent network activity.

### Pipe configuration

One pipe for the primary device VLAN with default bandwidth allocation.

### Queue configuration

| Queue | Priority | Purpose |
|-------|----------|---------|
| primary-queue | 100 (high) | Primary device traffic (gaming, streaming) |

### Firewall rules

Primary device VLAN traffic is routed through the queue:

```
Interface: Primary
Direction: inbound
Protocol: any
Source: Primary subnet
Queue: primary-queue
```

**Effect:** gaming and video streaming packets are deprioritized less than
background download or service traffic, maintaining responsiveness during
congestion.

---

## Firewall rules (current)

Basic per-VLAN allow rules:

| VLAN | Action | Source | Destination |
|------|--------|--------|-------------|
| Management | Allow | MGMT subnet | any |
| Primary | Allow | Primary subnet | any |
| Servers | Allow | Servers subnet | any |

**Security gap:** These rules are overly permissive. The Primary device can
currently reach the Servers VLAN, defeating segmentation.

### Hardening plan (to-do)

Replace the per-VLAN allow-all rules with specific policies:

1. Management → ad-blocking DNS (port 53) and firewall (web UI)
2. Primary → ad-blocking DNS (port 53) and WAN (internet)
3. Servers → ad-blocking DNS (port 53) and WAN (internet)
4. Block all inter-VLAN traffic by default

This requires careful testing — do one rule at a time and verify no lockouts.

---

## Hardening completed

- Encrypted config backups
- Full recursive DNS resolution (no third-party dependencies)
- DHCP service bound to correct interfaces
- QoS for performance
- Per-VLAN firewall rules (basic)

## Hardening planned

- Specific inter-VLAN firewall rules (top priority)
- Suricata IDS/IPS (detection mode)
- Web UI / SSH access restricted to management VLAN
- VPN integration for encrypted internet uplink

