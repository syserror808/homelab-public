# Troubleshooting & Lessons Learned (Public)

A collection of real infrastructure problems, their root causes, and solutions.
Kept as a reference for future issues and a portfolio of diagnostic methodology.

---

## Factory reset with old config on new hardware

**Problem:** After a factory reset and restoring a config backup from the old
firewall, the network was dead. Interfaces showed as "up" with valid IPs, but
traffic wouldn't flow (`ping` returned "Host is down").

**Root cause:** The old backup referenced hardware (NICs) that didn't exist on
the new appliance. OPNsense had the IP assigned but the interface was attached
to a non-existent physical port, so packets were dropped silently.

**Lesson:** After a hardware change, rebuild interface assignments from scratch
rather than relying on config backups. Hardware NIC naming differs between
devices.

---

## WAN and LAN on same subnet — no internet

**Problem:** WAN interface had a DHCP-assigned IP, LAN was set to the same subnet.
Internet was completely broken despite all configs looking valid.

**Root cause:** When WAN and LAN are on the same subnet, the firewall can't
distinguish which traffic should go to the internet and which should stay local.
NAT logic fails.

**Fix:** Changed LAN to a different subnet. Internet worked immediately.

**Lesson:** **WAN and LAN must be on different subnets.** This is a non-negotiable
requirement. Check subnet overlap first when diagnosing routing problems.

---

## DHCP service not listening on VLAN interfaces

**Problem:** DHCP ranges were configured for all three VLANs, but devices on
those VLANs got "No DHCPOFFERS received." Meanwhile, the primary LAN worked fine.

**Root cause:** The DHCP daemon (Dnsmasq) was configured to listen only on the
primary LAN interface. It literally didn't have a socket on the VLAN interfaces,
so DHCP requests on those ports were ignored.

**Fix:** In Dnsmasq configuration, explicitly added all VLAN interfaces to the
listening list:

```
Listening interfaces: LAN, VLAN-MGMT, VLAN-Primary, VLAN-Servers
```

DHCP worked immediately.

**Lesson:** This is a common misconfiguration. When DHCP ranges are defined but
not working, check that the DHCP daemon is actually listening on those
interfaces. Easy to overlook, causes multi-day outages.

---

## Container can't get IP — multi-layer problem

Installing a Pi-hole container on Proxmox uncovered four separate issues:

### Issue 1: Bridge not VLAN-aware
**Symptom:** Container only got IPv6 link-local, no IPv4.

**Fix:** Added `bridge-vlan-aware yes` to the bridge config.

### Issue 2: Double-tagged frames (container adding tag on untagged port)
**Symptom:** DHCP timeouts even after bridge fix.

**Root cause:** Switch port is untagged (PVID handles VLAN), but container tried
to add its own VLAN tag. Result: frames double-tagged, never reached OPNsense.

**Fix:** Removed `tag=` from container network interface. Switch already handles
VLAN tagging.

**Lesson:** untagged access port → no VLAN tag on the device. Tagged trunk port
→ tag on the device. Don't mix them.

### Issue 3: DHCP daemon not listening on VLAN (see above)

### Issue 4: DNS service only answering its own subnet
**Symptom:** Cross-VLAN DNS queries timed out.

**Fix:** Enabled "Permit all origins" in DNS settings.

---

## Realtek WAN NIC watchdog timeout (hardware fault)

**Problem:** During heavy downloads (>100 Mbps sustained), the internet connection
would drop. Speed would collapse, then slowly recover. Reproducible and consistent.

**Logs:** `re0: watchdog timeout` followed by interface reset.

**What was tested:**
- Cable (swapped, no change)
- Firewall rules (disabled, no change)
- Direct-to-modem bypass (stable, never timed out)
- Driver update (no fix)

**Conclusion:** Hardware fault. The Realtek RTL8211F NIC has a design issue or
defect causing it to drop under load.

**Fix:** Replaced firewall appliance with one using Intel NICs. Problem gone.

**Lesson:** Realtek NICs are known to be unreliable in always-on router/firewall
use. For production, insist on Intel. When you see watchdog timeouts, don't waste
time on driver updates — replace the hardware.

---

## Windows 11 on Windows 10-only platform

**Problem:** Management laptop upgraded to Windows 11. After the upgrade, external
display via HDMI stopped working. Windows reported "PC can't project to another
screen."

**Root cause:** Device Manager showed "Microsoft Basic Display Adapter" — the
generic fallback. The platform (Intel UHD 605, Pentium Silver N5000) has **no**
Windows 11 driver support. HP officially supports Windows 10 only.

**Lesson:** Before upgrading Windows, check the CPU/GPU against the manufacturer's
supported list. Older platforms often don't have Win11 drivers. The fallback
display adapter can't drive external displays.

**Fix (recommended):** Downgrade to Windows 10. Better driver support, better
performance on 4GB RAM.

---

## Gaming latency after network topology change

**Problem:** After moving the primary PC from a direct firewall connection to
going through a managed switch and VLAN, gaming latency increased noticeably.

**Root cause:** Without QoS, background network traffic (downloads, service
activity) was competing with gaming packets. No prioritization.

**Fix:** Deployed OPNsense traffic shaper:
1. Created a pipe for the primary device VLAN
2. Created a queue with high priority (weight 100)
3. Added a firewall rule routing the VLAN's traffic through the queue

Result: gaming latency returned to baseline.

**Lesson:** When moving devices into managed infrastructure, plan for QoS. Add
traffic shaping rules for latency-sensitive services (gaming, streaming, VoIP).

---

## General diagnostic methodology

**Layer-by-layer troubleshooting (in order):**

1. **Physical link** — `ethtool` or switch LEDs showing link up?
2. **IP assignment** — does device have an IP? `ip addr show`
3. **Local connectivity** — can it reach gateway? `ping <gateway>`
4. **DNS** — can it resolve names? `nslookup` vs `ping <raw_ip>`
5. **Service availability** — is the daemon listening? `netstat`, port scans

**Example:** "No internet"
- If `ping 8.8.8.8` works but `ping google.com` fails → DNS problem
- If both fail → routing problem
- If only local tests fail → physical layer issue

**Key tell:** If IP connectivity works (ping raw IPs) but DNS fails (nslookup),
it's **always** DNS. Don't waste time on routing.

