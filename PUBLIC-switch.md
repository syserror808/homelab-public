# Switch Configuration (Public)

802.1Q VLAN configuration for a managed 8-port switch.

---

## Port assignments

| Port | Connected to | VLAN | Tagged/Untagged | PVID |
|------|---|---|---|---|
| 1 | Firewall (trunk) | 10, 20, 30 | Tagged | 1 |
| 2 | Primary device | 20 | Untagged | 20 |
| 3 | Hypervisor primary | 30 | Untagged | 30 |
| 4 | Hypervisor backup | 30 | Untagged | 30 |
| 5–8 | Unused | (default) | Untagged | 1 |

---

## 802.1Q VLAN membership

| VLAN | Name | Tagged ports | Untagged ports |
|------|------|---|---|
| 1 | Default | — | Ports 1, 5–8 |
| 10 | Management | Port 1 | — |
| 20 | Primary | Port 1 | Port 2 |
| 30 | Servers | Port 1 | Ports 3–4 |

**Port 1 (firewall):** Carries multiple VLANs as **tagged** traffic (802.1Q headers
present). The firewall decodes the tags and routes accordingly.

**Ports 2–4 (end devices):** Untagged access ports. PVID tells the switch which
VLAN to assign incoming untagged frames. No 802.1Q headers needed on the devices.

---

## PVID configuration

| Port | Device | PVID | Notes |
|------|--------|------|-------|
| 1 | Firewall | 1 | Trunk port, handles tags |
| 2 | Primary device | 20 | All untagged frames → VLAN 20 |
| 3 | Hypervisor 1 | 30 | All untagged frames → VLAN 30 |
| 4 | Hypervisor 2 | 30 | All untagged frames → VLAN 30 |
| 5–8 | Unused | 1 | Default VLAN |

---

## Key concepts

**Tagged vs untagged:**
- **Tagged** — 802.1Q header is present, VLAN ID in the frame. Used by routers
  carrying multiple VLANs (trunk ports).
- **Untagged** — No VLAN header. Switch assigns the frame to the PVID. Used by
  end devices (servers, PCs).

**PVID (Port VLAN ID):**
- Determines which VLAN untagged frames on that port belong to
- Must match the device's actual VLAN
- Mismatch = traffic goes to the wrong VLAN, device loses connectivity

**Trunk port:**
- Tagged member of multiple VLANs
- Carries all VLAN traffic in one physical cable
- Used between switches or firewall + switch

**Access port:**
- Untagged member of a single VLAN
- Used for end devices (servers, desktops)
- Simpler, but can only carry one VLAN per port

---

## Configuration steps (general)

1. **Enable 802.1Q mode** on the switch
2. **Create VLAN entries** (VLAN 10, 20, 30)
3. **Add ports to VLANs** — specify tagged vs untagged
4. **Set PVID** on each port to match its VLAN
5. **Remove ports from unused VLANs** (e.g., remove untagged from VLAN 1 if now on VLAN 20)

---

## Common mistakes

- **Forgetting to remove from default VLAN.** A port left on both VLAN 1 and
  VLAN 20 can cause conflicts.
- **Tagged on an end device.** If the switch port is tagged (trunk) but the
  device doesn't understand 802.1Q, frames are dropped.
- **Untagged on a trunk port.** The firewall expects 802.1Q headers; untagged
  traffic may be dropped or assigned to the wrong VLAN.
- **PVID mismatch.** PVID 20 but the device is on VLAN 30 = disconnected.

**Lesson:** always double-check port membership and PVID after changes. Test
connectivity immediately.

