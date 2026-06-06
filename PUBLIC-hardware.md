# Hardware (Public)

Current and planned hardware with rationale for choices.

---

## Current inventory

| Device | Role | Notes |
|--------|------|-------|
| Enterprise-class appliance | Firewall/router | Intel networking CPUs, fanless |
| Server mini PC #1 | Proxmox hypervisor (primary) | Runs Pi-hole and future services |
| Server mini PC #2 | Proxmox hypervisor (backup) | Replication and high availability |
| Managed 8-port switch | Network switching | 802.1Q VLAN support |
| Desktop PC | Daily driver | Gaming/streaming, VLAN 20, QoS prioritized |
| Management laptop | Planned console | Currently blocked on OS driver issues |

---

## Firewall choice — Enterprise appliance

Selected an enterprise-class networking appliance over commercial consumer routers
for several reasons:

**Why not consumer routers?**
- Limited VLAN support (often none)
- No real firewall rules engine
- Limited routing options
- Can't run open-source OS

**Why enterprise appliance?**
- Multiple Intel GbE ports (originally had Realtek NICs that had watchdog timeout
  issues under load)
- Can run OPNsense (open-source, well-documented)
- Full VLAN support and firewall rule engine
- Quiet (fanless), low power consumption (~20W)
- Better long-term support and community

**Trade-off:** More complex to configure initially, but full control and stability
once set up.

---

## Hypervisor choice — Proxmox

Two server mini PCs running Proxmox CE (community edition) for:

- **Clustering:** two nodes can replicate VMs/containers
- **Live migration:** stop VMs without downtime (theory; haven't tested)
- **Container native:** LXC is lightweight and fast
- **Open source:** full control, no licensing costs
- **Active community:** good docs and forums

**Future:** Consider adding storage cluster (Ceph) when a NAS arrives.

---

## Network appliance choice — Managed switch

Managed switch (vs unmanaged) because:

- Full 802.1Q VLAN support (required for this design)
- Port-based config, no shared backbone
- QoS options for traffic prioritization
- Affordable (~$30–50 for 8-port managed switch)

---

## DNS/ad-blocking — Pi-hole

Chose Pi-hole over other ad-blockers because:

- Runs as a container (no additional hardware)
- Network-wide blocking (all devices, all VLANs)
- Nice dashboard + query logs
- Can forward to recursive resolver (Unbound)
- Single point of failure — documented risk, secondary Pi-hole planned

---

## Planned upgrades

| Item | Purpose | Rationale |
|---|---|---|
| NAS | Backups + media | Storage is the bottleneck, not compute |
| Secondary Pi-hole | DNS redundancy | Primary Pi-hole is a SPoF |
| GPU for LLM | Ollama inference | Local AI without cloud API calls |

---

## Hardware lessons

**Realtek WAN NIC failure:** The original firewall had a Realtek RTL8211F NIC that
showed watchdog timeouts under sustained load (downloads >100 Mbps). Logs:
`re0: watchdog timeout`. No driver update fixed it. Lesson: Realtek NICs are
unreliable for always-on firewall/router use. Insist on Intel for production.

**Windows 11 on Windows 10-only platform:** The management laptop has an Intel
UHD 605 (Pentium Silver N5000) with no Windows 11 driver support. After
upgrading to Win11, external display (HDMI) stopped working — no driver to speak
to the GPU. Lesson: check hardware compatibility before OS upgrades, especially
on older CPUs.

---

## Capacity notes

- **Firewall:** plenty of headroom (Intel Atom C3558 is overkill for home use)
- **Hypervisors:** mini PCs have 8GB RAM, ample for light workloads (Pi-hole, Docker)
- **Network:** 8-port switch is fine for a home lab; scalable up to larger switch if needed
- **Storage:** currently none; waiting for NAS prices to drop

