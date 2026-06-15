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

Handles DHCP and local DNS for all VLANs. Must be explicitly bound to each
VLAN interface. Static DHCP reservations by MAC address lock key hosts to
fixed IPs so NAT rules and DNS config never silently break.

DHCP option 6 on all VLANs points to Pi-hole, ensuring all clients use the
filtering resolver regardless of VLAN.

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

Firewall sits behind ISP router. Standard port forwarding has no effect until
the ISP router treats the firewall as the true inbound edge.

Solution: ISP router Static NAT → firewall WAN IP. Avoids bridge mode which
would have disabled the ISP router's DHCP/WiFi for other household devices.

### Outbound NAT — static port rule for gaming

Prevents the firewall from randomizing UDP source ports for the gaming machine.
Port randomization breaks NAT type negotiation in games using UDP hole-punching.
Result: NAT Strict → Moderate.

### Destination NAT — gaming ports

Port forwards for relevant UDP gaming ports to the gaming machine's reserved IP.

Critical detail: the firewall rule association field must be set to
**Register rule** — not Manual or Pass. This auto-generates the required
associated firewall pass rule. Without it the Destination NAT rule exists in
the UI but is silently skipped. Result: NAT Moderate → Open.

---

## Firewall rules

### VLAN 10 — MGMT

Full outbound access. Administration VLAN — no restrictions on what it can reach.

### VLAN 20 — Workstation (hardened 2026-06-15)

**Key lesson learned:** The DNS allow rule must be placed above the Server VLAN
block rule. Pi-hole lives on the Server VLAN subnet — if the block rule matches
first, DNS breaks even though there's an explicit allow rule for it. OPNsense
processes rules top-down, first match wins.

| Order | Action | Destination | Port | Description |
|---|---|---|---|---|
| 1 | Allow | Pi-hole IP | 53 | DNS to Pi-hole |
| 2 | Block | MGMT subnet | any | Block → MGMT |
| 3 | Block | Server subnet | any | Block → Server |
| 4 | Allow | any | any | Allow internet |

Verified: cross-VLAN pings fail, internet and DNS work, Open NAT intact.

### VLAN 30 — Server (hardened 2026-06-15)

Same DNS-first lesson applies: Pi-hole needs to reach Unbound on the MGMT
gateway IP for recursive resolution. DNS allow must be above the MGMT block.

| Order | Action | Destination | Port | Description |
|---|---|---|---|---|
| 1 | Allow | MGMT gateway | 53 | DNS to Unbound |
| 2 | Block | Workstation subnet | any | Block → Workstation |
| 3 | Block | MGMT subnet | any | Block → MGMT |
| 4 | Allow | any | any | Allow internet |

Verified: Pi-hole DNS chain intact, Server cannot reach Workstation or MGMT.

---

## Management UI restriction (two-layer)

### Service-level binding
- OPNsense: listen interface set to MGMT VLAN only
- Proxmox: source-IP filter via pveproxy ALLOW_FROM
- Pi-hole: lighttpd bound to Server VLAN IP

### Firewall block rules
Block rules on Workstation and Server VLAN interfaces targeting admin service
ports (OPNsense web UI, Proxmox 8006, Pi-hole web UI). Rules sit above
allow rules so they match first.

---

## Suricata IDS

- Capture mode: netmap (alert-only)
- Pattern matcher: Hyperscan, promiscuous mode enabled
- Active rulesets: ET Open core (malware, exploit, botcc, phishing),
  abuse.ch feeds, TOR exit nodes
- Excluded: gaming rulesets (false positives), P2P rulesets (VPN conflict)
- Enforcement: OPNsense default deny; Suricata provides visibility only
