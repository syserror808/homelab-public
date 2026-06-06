# Project Status (Public)

Current state and next steps for the homelab infrastructure.

---

## What's working

- **Core network:** OPNsense firewall routing, VLAN segmentation, DHCP and DNS services
- **Encrypted DNS** — DNS-over-TLS, DNSSEC, full recursive resolution
- **Encrypted backups** — all configurations backed up with encryption
- **Hypervisor cluster** — two server nodes running Proxmox, connected for replication
- **Pi-hole service** — network-wide ad-blocking DNS, deployed as LXC container
- **Automatic DNS failover** — DHCP delivers Pi-hole as primary, firewall as secondary
- **Primary device** — QoS configured to maintain gaming/streaming performance during congestion
- **Backup automation** — Proxmox Backup Server connected and tested

---

## Recent achievements (June 2026)

**Infrastructure rebuild:** Complete factory reset of firewall and VLAN infrastructure
from scratch. The session involved:

1. Clean OPNsense installation and interface assignment
2. VLAN trunk and access port configuration
3. Separate subnet assignment for WAN and LAN (critical for NAT)
4. DHCP service binding to all VLAN interfaces
5. Two hypervisor deployments on VLAN 30
6. Fresh Pi-hole installation and network-wide DNS integration
7. QoS traffic shaper configuration for performance optimization

**Key problems solved:**
- Firewall interface assignment conflicts (old config vs new hardware)
- WAN/LAN subnet collision preventing NAT
- DHCP service not listening on VLAN interfaces
- DNS-over-TLS configuration and verification
- Gaming latency issues (fixed with QoS)

All services now stable and tested.

---

## Next priorities

### Security hardening
- **Firewall rule refinement** — Replace allow-all rules with specific per-VLAN policies
  - Allow DNS to Pi-hole only
  - Allow outbound internet (WAN) only
  - Block all inter-VLAN traffic by default
- **Secondary DNS fallback** — Add firewall as backup DNS in DHCP if Pi-hole is down
- **IDS/IPS** — Deploy Suricata in detection mode

### Service expansion
- **Container orchestration** — Docker host for light services (monitoring, dashboards)
- **GPU passthrough** — Ollama for local LLM inference (requires IOMMU/VFIO)
- **Redundancy** — Secondary Pi-hole container on backup node for DNS failover

### Deferred (waiting for storage)
- **NAS deployment** — Local backup storage and media server foundation
- **Media services** — Jellyfin or Plex, Immich for photo backups
- **VPN services** — Mullvad integration on firewall (latency tradeoff)
- **Inbound access** — WireGuard for remote management

---

## Infrastructure notes

**VLAN design:** Three-tier segmentation — management, primary device, and servers.
Each VLAN has its own subnet and firewall rules.

**Firewall architecture:** Single-point-of-failure OPNsense with all routing logic
centralized. Future: consider backup firewall.

**DNS resilience:** Pi-hole blocks ads; Unbound provides full recursive resolution.
If Pi-hole is down, devices can still resolve via firewall (with ads). A secondary
Pi-hole on the backup node is planned.

**Double-NAT:** Internet service provider's modem has no bridge mode, resulting
in double-NAT. Functional but limits inbound access. Port-forwarding for services
would require rules on both NAT layers.

**Performance tuning:** QoS traffic shaper prioritizes primary device traffic to
maintain gaming/streaming responsiveness during background downloads or service activity.

