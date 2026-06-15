# Hardware

---

## Active hardware

### Firewall / router — dedicated appliance

- Intel NICs (dual-port)
- Acquired used (~$135)
- Replaces: original unit with Realtek NIC that failed under sustained load

Choosing Intel over Realtek for firewall NICs is a hard lesson from this build —
documented in the troubleshooting log.

### Hypervisor — mini PC (primary)

- Proxmox VE
- Hosts Pi-hole LXC and planned future services

### Management workstation — HP Pavilion x360 (Pentium Silver)

- Connected via KVM switch to MGMT VLAN port
- Display quirks: Win10 Intel graphics driver force-installed on Win11;
  KVM passthrough non-functional due to KVM lacking EDID emulation
- Planned replacement: compact business desktop (ThinkCentre-class) which
  avoids the EDID/driver issues entirely

### Primary workstation (gaming/daily driver)

- Static DHCP reservation by MAC address
- Holds the Open NAT configuration for gaming
- On isolated workstation VLAN

### Managed switch

- 802.1Q VLAN support
- Trunk port to firewall carries tagged traffic for all VLANs
- Access ports per VLAN for end devices

---

## Retired hardware

- Original mini PC with Realtek NIC: failed under sustained throughput.
  Candidate for repurposing as secondary Pi-hole (DNS redundancy).

---

## Planned

| Item | Purpose |
|---|---|
| Compact business desktop (ThinkCentre-class) | Dedicated MGMT machine — eliminates KVM/display issues |
| NAS | Proxmox backup target + media/photo storage |

