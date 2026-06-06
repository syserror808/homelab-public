# Homelab - Public Documentation

Portfolio documentation for a home network and server infrastructure. This repo
demonstrates network architecture, Linux systems administration, virtualization,
and infrastructure troubleshooting.

> This is the **public-facing version** with sensitive details sanitized.

---

## Architecture Overview

A segmented home network with VLAN isolation, managed firewall, and virtualized
server infrastructure:

- **Firewall/router** — OPNsense on dedicated hardware
- **Switch** — Managed, 802.1Q VLAN capable
- **Hypervisor cluster** — Two server nodes running Proxmox (primary + backup)
- **Primary PC** — Daily driver with QoS prioritization
- **Services** — Pi-hole (ad-blocking DNS), Proxmox Backup Server

The network is segmented into three VLANs for isolation and security. All
traffic is routed through the OPNsense firewall with per-VLAN rules.

---

## Documentation

| File | Topic |
|------|-------|
| [status.md](status.md) | Current state and next steps |
| [network/diagram.md](network/diagram.md) | Network topology (sanitized) |
| [network/opnsense.md](network/opnsense.md) | Firewall config — VLANs, DHCP, DNS, QoS |
| [network/switch.md](network/switch.md) | 802.1Q VLAN configuration |
| [proxmox.md](proxmox.md) | Hypervisor hosts and services |
| [hardware.md](hardware.md) | Hardware choices and rationale |
| [troubleshooting.md](troubleshooting.md) | Problems solved and lessons learned |

---

## Key skills demonstrated

- **Network segmentation** — VLAN design and 802.1Q configuration
- **Firewall administration** — OPNsense routing, rules, and QoS
- **DNS & DHCP** — Unbound recursive resolution, Dnsmasq DHCP, Pi-hole integration
- **Linux systems** — Proxmox, LXC containers, networking stacks
- **Virtualization** — Proxmox clustering, VM/container management, backups
- **Infrastructure troubleshooting** — layer-by-layer diagnosis, root cause analysis
- **Documentation** — comprehensive network design and operational runbooks

---

## Architecture highlights

**Network isolation:** Three VLANs (management, primary device, servers) with
firewall rules enforcing segmentation.

**Encrypted DNS:** Full DNS-over-TLS with DNSSEC and recursive resolution in-house.

**Ad-blocking:** Network-wide ad blocking via Pi-hole with DNS policy enforcement
across all VLANs via DHCP.

**High availability:** Primary and backup hypervisor nodes with automated backup
to dedicated Proxmox Backup Server.

**Performance optimization:** QoS traffic shaping to maintain gaming/streaming
performance on the primary device despite network congestion.

---

## Build timeline

- **Phase 1:** Factory reset and VLAN infrastructure rebuild (June 2026)
- **Phase 2:** Proxmox cluster deployment and Pi-hole integration (June 2026)
- **Phase 3:** QoS optimization and service layer hardening (June 2026)
- **Phase 4:** Planned — NAS deployment, Ollama GPU passthrough, additional services

---

## Lessons learned

See [troubleshooting.md](troubleshooting.md) for detailed case studies on:
- VLAN interface configuration gotchas
- DNS/DHCP service binding issues
- Hardware compatibility and watchdog timeouts
- Network layering and diagnostic methodology
- Operating system driver support verification

