# OPNsense Configuration

Hardware: Dedicated x86 firewall appliance (Intel NICs)
WAN: Residential fiber ISP (double-NAT environment)
Interfaces: WAN + three VLAN subinterfaces

---

## VLAN interfaces

| VLAN | Name | Subnet | Purpose |
|---|---|---|---|
| 10 | MGMT | 10.0.10.0/24 | Management and admin workstations |
| 20 | Workstation | 10.0.20.0/24 | Primary gaming/workstation PC |
| 30 | Server | 10.0.30.0/24 | Proxmox, Pi-hole, self-hosted services |

---

## DNS / DHCP

### Dnsmasq

Handles DHCP and local DNS for all VLANs. Must be bound explicitly to each
VLAN interface — not bound by default.

Static DHCP reservations by MAC address ensure key hosts (gaming PC, DNS server)
hold fixed IPs so NAT rules and DNS configurations never silently break.

DHCP option 6 (DNS server) on all VLANs points to Pi-hole, ensuring all clients
use the filtering resolver.

### DNS chain

```
clients
  ↓
Pi-hole — ad/tracker filtering
  ↓
Unbound — recursive resolver, DNSSEC enforced
  ↓
root servers (no third-party forwarders)
```

---

## NAT — double-NAT environment

The firewall sits behind an ISP-provided router. The firewall's WAN IP is a
private address, not the public IP. Standard port forwarding in OPNsense has no
effect until the ISP router is configured to treat the firewall as the true edge.

**Solution:** ISP router Static NAT pointing all inbound traffic to the
firewall's WAN IP. This avoids the need for bridge mode (which on this ISP
device would have disabled DHCP for other household devices).

**Side effect:** With the firewall now receiving all inbound internet traffic,
Suricata starts seeing internet scan/probe traffic that the ISP router was
previously absorbing silently. This is correct and expected behavior.

### Outbound NAT (hybrid mode) — gaming PC

A static port rule prevents OPNsense from randomizing UDP source ports for the
gaming PC's IP. Port randomization causes NAT type degradation in games that
use UDP hole-punching for peer negotiation.

### Destination NAT — gaming ports

Port forwards for the relevant UDP gaming ports pointing to the gaming PC's
reserved IP.

Critical configuration detail: the "firewall rule association" field in
OPNsense Destination NAT must be set to **Register rule** (not Manual or Pass).
This auto-generates the associated firewall pass rule. Without it, the
Destination NAT rule exists but is silently skipped — the ports remain
effectively closed despite the rule appearing correct in the UI.

Result: Open NAT from a double-NAT setup, no DMZ, no bridge mode.

---

## Firewall rules

### Management UI restriction

Two-layer approach:

**Service-level binding:**
- OPNsense: listen interface set to MGMT VLAN only
- Proxmox: source-IP filter in pveproxy configuration
- Pi-hole: lighttpd bound to server VLAN IP

**OPNsense block rules** (on workstation and server VLAN interfaces):
Block rules placed above any allow rules targeting admin service ports
(OPNsense web UI, Proxmox port 8006, Pi-hole web UI). OPNsense processes
rules top-down; block rules must be above allow-all rules to be effective.

### Workstation VLAN — pending hardening

Current state: block rules above a legacy allow-all. Inter-VLAN traffic is
partially restricted but not fully segmented.

Target state: replace allow-all with explicit specific rules:
- Allow → Pi-hole (DNS port 53)
- Allow → internet (WAN)
- Block → MGMT VLAN
- Block → Server VLAN

---

## Suricata IDS

- Capture mode: netmap (alert-only; no inline blocking)
- Pattern matcher: Hyperscan, promiscuous mode enabled
- Active rulesets: ET Open core (malware, exploit, botcc, phishing), abuse.ch
  botnet/feodo/ssl feeds, TOR exit nodes
- Excluded: gaming rulesets (false positives on game traffic), P2P rulesets
  (conflict with planned VPN routing)
- Enforcement: OPNsense default deny handles blocking; Suricata provides
  visibility only

