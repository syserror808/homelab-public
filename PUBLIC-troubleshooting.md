# Troubleshooting Log

Real problems encountered during this build, with root-cause analysis and fixes.
Includes dead-ends — the failed paths are often more instructive than the fixes.

---

## Problem 1 — Intermittent internet that wasn't the WAN

**Symptoms:** Browsing and apps failed intermittently. Downloads stalled
partway. Everything pointed to a flapping WAN connection.

**Initial investigation (wrong direction):** Logs showed an occasional NIC
error on the firewall, so investigation focused there. Cables swapped, NIC
driver updated — neither helped. The intermittent pattern continued.

**What it actually was:** DNS resolution was failing. The recursive resolver had
been left disabled, and the DHCP/local-DNS service was handling DHCP only, not
resolving names. Cached DNS entries worked until they expired, then lookups
failed — which produced the "works, then stops" pattern.

**The diagnostic key:** `ping <IP>` worked. `ping <hostname>` failed. That
combination always means DNS, not connectivity.

**Fix:** Re-enabled the recursive resolver.

**Lesson:** If raw IP pings succeed but name resolution fails, don't chase the
WAN or hardware. Check the resolver stack first.

---

## Problem 2 — Hardware NIC failure under load

**Symptoms:** Internet dropped reproducibly under sustained high-throughput
load (e.g. a large download). Recoverable — the link would reset and come back.

**Root cause:** The firewall's onboard NIC (Realtek chipset) was faulty. Logs
showed a watchdog timeout error followed by link-down and driver reset. The NIC
stops responding under sustained throughput and the driver resets it to recover.

**Tried and failed:** Vendor NIC driver plugin, cable swap, different switch port.

**Fix:** Replaced the firewall hardware with a unit using Intel NICs. No
instability since.

**Lesson:** A reproducible watchdog timeout in NIC logs is a hardware verdict.
Realtek NICs are a well-known weak point for sustained-throughput firewall use.
Intel NICs (i210, i211, i225, i226) are the community standard for this role.

---

## Problem 3 — DHCP silent on VLANs (weeks of static IPs as a workaround)

**Symptoms:** Devices on secondary VLANs couldn't get DHCP leases. This had
been accepted as "normal" and worked around with manual static IPs.

**Root cause:** The DHCP/DNS service on OPNsense was bound to only one
interface. DHCP requests arriving on other VLAN interfaces were never seen by
the service and got no response.

**Fix:** Explicitly added all VLAN interfaces to the service's listen list.
DHCP worked immediately on all VLANs. Devices that had been statically
configured were switched back to DHCP.

**Lesson:** Interface binding in DHCP/DNS services is not automatic. If DHCP is
silent on a VLAN, verify that the service is actually listening on that
interface before chasing anything else.

---

## Problem 4 — LXC container: four stacked networking failures

**Symptoms:** A Proxmox LXC container for network-wide DNS filtering couldn't
obtain an IP or reach the network after deployment.

**Root causes (all four had to be fixed):**

1. **Proxmox host bridge missing VLAN awareness.** The host's Linux bridge
   needed `bridge-vlan-aware yes` and `bridge-vids 2-4094` in the network
   config before it could pass 802.1Q tagged traffic to containers.

2. **Incorrect VLAN tag on the container interface.** A `tag=X` setting was
   added to the container's network config — but the switch port was
   untagged for that VLAN, so the container should not tag its own traffic.
   Setting a tag when the upstream switch port is untagged causes a mismatch.

3. **Missing DHCP setting.** During config edits, `ip=dhcp` was accidentally
   deleted. The container had no address acquisition method at all.

4. **DHCP service not listening on the VLAN.** Even after fixes 1–3, DHCP
   responses never arrived because the DHCP service had the interface binding
   problem described in Problem 3.

**Lesson:** Container networking failures can stack across the host bridge, the
container config, and the upstream router simultaneously. Debug methodically
layer by layer: host bridge → container interface config → DHCP server.

---

## Problem 5 — Gaming Strict NAT behind double-NAT (ISP router + firewall)

**Symptoms:** Game client reporting Strict NAT despite port forwards in the
firewall. Poor matchmaking, connectivity issues.

**Initial attempts (failed):**
- UPnP plugin on the firewall — no effect
- Standard Destination NAT port forwards — present and correct, but no change
  to reported NAT type

**Root cause:** The firewall was not the true WAN edge. An ISP-provided router
sat in front of it, meaning the firewall's WAN IP was a private address, not
the public IP. NAT traversal happens at the ISP router layer first — the
firewall's port forwards never saw the relevant traffic.

**Failed path: ISP router bridge mode**
Attempted to enable bridge mode on the ISP router to make the firewall the
true edge. Enabling the option caused a complete internet outage for all
household devices — the router's DHCP also stopped serving when bridge mode
was activated. Recovery required a factory reset (done twice). This device's
bridge mode is all-or-nothing and incompatible with a shared-household setup.

**Solution (three-part):**

1. **ISP router Static NAT** — point all inbound traffic to the firewall's WAN
   IP. This makes the firewall the effective inbound edge without requiring
   bridge mode or disrupting other household devices.

2. **Outbound NAT static port rule** — in hybrid mode, add a rule that
   prevents the firewall from randomizing UDP source ports for the gaming
   machine's IP. Port randomization breaks NAT type negotiation for UDP games.
   Moved NAT: Strict → Moderate.

3. **Destination NAT with correct rule association** — port forwards on the
   relevant UDP ports to the gaming machine's reserved IP. The critical detail:
   the rule association field must be set to auto-register an associated
   firewall pass rule. With the wrong association setting, the Destination NAT
   rule exists in the UI but is silently skipped.
   Moved NAT: Moderate → Open.

**Also required:** A static DHCP reservation for the gaming machine's MAC
address. Without it, the machine could get a different IP on next lease renewal,
silently breaking all the NAT rules.

**Lesson 1:** Port forwards have no effect if the firewall is not the true WAN
edge. Confirm the firewall's WAN IP is the public IP before debugging NAT rules.

**Lesson 2:** In OPNsense Destination NAT, the firewall rule association field
must be set to "Register rule" to auto-generate the associated pass rule.
"Manual" leaves the pass rule absent and the forward silently fails.

---

## Problem 6 — Proxmox web UI 401 despite correct credentials

**Symptoms:** Proxmox web UI returned 401 Unauthorized. Correct root password,
Linux PAM realm selected. No helpful error message.

**Root cause:** PAM account lockout after repeated failed login attempts. PAM
doesn't distinguish between "wrong password" and "account locked" in its
response — both return 401.

**Fix:** At the Proxmox console (not web UI):
```bash
pam_tally2 --user root --reset
```

**Lesson:** When a correct password is rejected without explanation, check
for account lockout before assuming the password is wrong.

---

## Problem 7 — Management laptop: no external display, then broken KVM passthrough

**Symptoms:** HP laptop used as management workstation had no external display
output. HP's driver page showed no applicable graphics driver.

**Root cause:** HP only released Windows 10 drivers for this hardware model.
The laptop was running Windows 11, which has no graphics driver available from
HP. No Win11 driver = no display output.

**Fix:** Set HP driver page to Windows 10 OS selection, downloaded the Win10
Intel graphics driver, force-installed it on Win11 via Device Manager. Direct
HDMI to monitor then worked.

**Dead-end: KVM passthrough**
After the driver fix, display through the KVM switch still failed. Investigation
found the KVM's spec sheet explicitly states it does not support EDID emulation.
The GPU on this laptop requires a valid EDID signal to output — unlike discrete
desktop GPUs, which tolerate a missing or generic EDID. No fix available short
of replacing the KVM or the machine. Set aside.

**Lesson 1:** When deploying a laptop on an unsupported OS, driver gaps are
likely. For a dedicated management machine, using the vendor-supported OS
eliminates these problems entirely.

**Lesson 2:** EDID emulation is not universal across KVM switches. Integrated
laptop GPUs tend to be strict about requiring a valid EDID. Verify KVM specs
before purchasing for use with a laptop console.

