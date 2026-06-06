# Network Diagram (Public)

Sanitized network topology and addressing scheme.

---

## Physical topology

```
Internet
   |
ISP modem/router
   |
   └─→ [WAN] OPNsense firewall
          |
          └─→ [VLAN trunk] GE port
             |
             └─→ Managed switch
                |
    ┌───────────┼───────────┬───────────┬─────────┐
    |           |           |           |         |
Port 2        Port 3      Port 4      Port 5   Ports 6-8
(untagged)    (untagged)  (untagged)  (unused) (unused)
  |             |           |
Primary PC   Hypervisor   Hypervisor  
             (primary)    (backup)
             - Pi-hole LXC
             - Services
```

---

## VLAN design

| VLAN | Name | Purpose | Subnet |
|------|------|---------|--------|
| 10 | Management | Admin/console access | 10.0.0.0/24 |
| 20 | Primary | Desktop/primary device | 10.0.1.0/24 |
| 30 | Servers | Hypervisors & services | 10.0.2.0/24 |

Each VLAN has its own subnet and firewall rules. Devices on different VLANs
require explicit firewall rules to communicate.

---

## Key IP assignments (sanitized)

| Device/service | Subnet | Notes |
|---|---|---|
| Firewall (MGMT) | 10.0.0.1 | Gateway for VLAN 10 |
| Firewall (Primary) | 10.0.1.1 | Gateway for VLAN 20 |
| Firewall (Servers) | 10.0.2.1 | Gateway for VLAN 30 |
| Primary PC | 10.0.1.x (DHCP) | Gets Pi-hole as DNS |
| Hypervisor primary | 10.0.2.10 (static) | Main compute node |
| Hypervisor backup | 10.0.2.11 (static) | Backup/replication |
| Pi-hole service | 10.0.2.50 (reserved) | DNS filtering |

DHCP pools: .100–.200 on each VLAN. Reserved static addresses kept outside
the pools to avoid lease collisions.

---

## Switch configuration

Standard managed switch with 802.1Q VLAN support.

| Port | Device | VLAN membership | PVID |
|------|--------|-----------------|------|
| 1 | Firewall trunk | 10, 20, 30 (tagged) | 1 |
| 2 | Primary device | 20 (untagged) | 20 |
| 3 | Hypervisor primary | 30 (untagged) | 30 |
| 4 | Hypervisor backup | 30 (untagged) | 30 |
| 5–8 | Unused | (default) | 1 |

**Key principle:** Firewall trunk port is tagged (carries multiple VLANs with
802.1Q headers), while end-device ports are untagged access ports (VLAN
membership set via PVID).

---

## WAN uplink notes

- ISP provides double-NAT (modem + firewall)
- OPNsense WAN gets a private IP address via DHCP from the modem
- This is functional but limits inbound access (port-forwarding would need
  rules on both NAT layers)
- Acceptable for a home lab; future could add bridge mode or bypass if available

