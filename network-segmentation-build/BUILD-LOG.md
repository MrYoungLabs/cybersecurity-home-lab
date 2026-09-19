# Build log

A record of the build in the order it happened, including the parts that did not
work. The failures are the useful content here. Each entry states the symptom,
the diagnosis and the fix, because a symptom without a diagnosis is just a
complaint.

---

## Phase 1: Hypervisor

**Goal:** Proxmox VE on bare metal, reachable over the network.

### Proxmox installer would not boot

**Symptom:** The USB installer was not offered as a boot device.

**Diagnosis:** Secure Boot was enabled with pre-enrolled EFI keys, so the
unsigned installer was refused before it ever ran.

**Fix:** Cleared the pre-enrolled keys in BIOS. The installer booted on the next
attempt.

### No video output after install

**Symptom:** The machine posted but produced nothing on the display.

**Fix:** Resolved at the BIOS display settings level. Worth noting because the
instinct in the moment is to assume the install failed, when the install was
fine and the output was going somewhere else.

### Host had no working network path

**Symptom:** Proxmox installed but was unreachable, with an interface configured
on a subnet nothing else lived on.

**Diagnosis:** The installer had picked up an interface and address that did not
match the actual cabling.

**Fix:** Edited `/etc/network/interfaces` directly, corrected the bridge and
address, and confirmed with `ip a` and `arp` from another host.

### Package updates failed

**Symptom:** `apt` errors on every update.

**Diagnosis:** Proxmox ships with the enterprise repository enabled, which
requires a subscription.

**Fix:** Disabled the enterprise repository and added `pve-no-subscription`.

**Outcome of phase 1:** Proxmox VE 9.2.2 running and reachable.

---

## Phase 2: Firewall as a virtual machine

**Goal:** pfSense running as VM 100 with real NICs, not bridged interfaces.

**Design decision:** pfSense could have gone on bare metal. Running it as a VM
on the same host as the lab targets consolidates everything onto one machine and
one power draw. The trade-off is that the whole network depends on the
hypervisor staying up, which is addressed below.

### Passthrough knocked the host off the network

**Symptom:** After passing all four NICs through to the pfSense VM, Proxmox
itself became unreachable.

**Diagnosis:** Obvious in hindsight. Every physical interface now belonged to the
guest, so the host had nothing left to talk on.

**Fix:** Returned PCI device 02 to the host and passed only devices 03 and 04 to
the VM. This is why ETH0 is reserved for Proxmox management in the port map.

### Could not log into pfSense with a known-correct password

**Symptom:** Console login rejected a password that was definitely right.

**Diagnosis:** A dead key on the laptop keyboard. The password being typed was
not the password being entered.

**Fix:** Different keyboard. Recorded here because a lot of time went into
suspecting the software before suspecting the hardware, and the lesson is that
"the input is correct" is an assumption like any other.

### Locked out of the web interface

**Symptom:** After tightening WAN rules, the pfSense GUI became unreachable.

**Diagnosis:** Administration was still happening from the WAN side, through a
temporary allow-all rule that had just been removed.

**Fix:** Established a path in from the LAN side first, at 192.168.30.1, then
deleted the temporary WAN rule and verified the GUI was no longer reachable from
the old network. Order of operations matters: build the new access path before
removing the old one.

### Network did not come back after a host reboot

**Symptom:** Rebooting Proxmox took the entire network down and it stayed down.

**Diagnosis:** VM 100 was not configured to start at boot, so the router simply
did not exist after a restart.

**Fix:** Enabled start at boot on VM 100. This is the mitigation for the
single-host design decision above, and it is the first thing worth checking in
any setup where the router is a guest.

---

## Phase 3: First VLAN and the trunk

**Goal:** One tagged VLAN carried from pfSense to the switch and out to a client.

**Design decision:** pfSense's LAN interface was set to IPv4 `None`, so `igc0`
carries no untagged address and functions as a pure 802.1Q trunk. Every VLAN is
a child interface on top of it. This keeps the trunk honest and avoids an
accidental untagged network.

### Could not get into the switch

**Symptom:** Web and SSH credentials for the Catalyst could not be recalled.

**Fix:** Console access over the switch's USB port, which appears as a COM port
under Ports in Device Manager, at 9600 baud. Recovered the local username with:

```
show run | include username
```

and wrote it down this time.

### Recurring TFTP errors on the console

**Diagnosis:** The switch was attempting a periodic configuration fetch from a
TFTP server that does not exist here.

**Fix:** `no service config`.

**Verification:** A laptop on an access port in VLAN 30 pulled 192.168.30.101
from pfSense DHCP across the tagged trunk. End to end, first VLAN working.

---

## Phase 4: Scaling to five segments

VLANs 40, 50, 60 and 70 were added the same way on both sides: created on
pfSense with an address and a DHCP scope, created on the switch with a name, and
added to the allowed list on the trunk.

**Design decision on the block rule.** The original lab rule blocked traffic to
`192.168.1.0/24` specifically. That was rewritten to block all of
`192.168.0.0/16` so that any VLAN created later is denied from the lab by
default rather than silently permitted. The failure mode of a segmentation rule
should be "too tight", not "too loose".

That decision is correct, and it is also what caused the hardest problem in the
whole build. See phase 7.

---

## Phase 5: Moving the management plane

**Goal:** Get Proxmox off the old household network and onto VLAN 40.

**The ordering problem:** Once Proxmox moves to VLAN 40 it only answers on VLAN
40, so the interface used to make the change stops working the moment the change
is applied. There is no way to do this remotely from the old network.

**Fix:** Set the new address at the physical console with a display and keyboard
attached, then change the switch port to VLAN 40 and reconnect from a host that
is already on that segment.

Proxmox now sits at 192.168.40.10 with its gateway at 192.168.40.1, on a segment
with no wireless presence at all.

---

## Phase 6: Wireless

**Goal:** A UniFi controller, an adopted access point, and SSIDs mapped to VLANs.

### PoE injector appeared dead

**Symptom:** The access point would not power on.

**Diagnosis:** The cable was in the data jack rather than the PoE output jack.

**Fix:** Correct jack. Also confirmed the injector was active 802.3bt rather than
passive. Passive injectors energise the pairs without a handshake and will damage
non-PoE hardware, and some vendors have historically shipped passive injectors
with their gear.

### UniFi would not install on the Pi

**Symptom:** The package repository returned nothing.

**Fix:** Corrected the repository configuration and installed UniFi Network
Application 10.6.101.

**Capacity note:** A 2 GB Pi 5 is below comfortable for this stack. MongoDB wants
roughly a gigabyte on its own and the Java application on top wants about as
much again. It runs, but moving the controller to a small VM on Proxmox is a
tracked open item.

### Locked out of the controller

**Symptom:** No working administrator credentials for the UniFi controller.

**Fix:** Password recovery directly against the controller's MongoDB backing
store with `mongosh`. Worth understanding as a security property in its own
right: anyone with local access to the controller host owns the controller, so
the host belongs on the management VLAN and not somewhere casual.

### Access point would not adopt

**Symptom:** The AP powered up and passed traffic, but never appeared in the
controller.

**Diagnosis:** An access point needs an untagged management address for itself.
Untagged traffic on a trunk falls into the port's native VLAN, which was VLAN 1.
VLAN 1 has no gateway and no DHCP server in this design, so the AP had no address
and could not find the controller.

**Fix:**

```
interface gi1/0/15
 switchport trunk native vlan 40
```

The AP picked up a management address on VLAN 40 and adopted.

### Test results that were not what they appeared to be

**Symptom:** A laptop reported working internet on the lab SSID, then failed
gateway pings.

**Diagnosis:** Windows had silently fallen back to the saved household SSID. The
traffic that worked was going over the old network entirely.

**Fix:** Forget the old SSIDs before testing, not after. A test that can
accidentally pass through the thing you are trying to bypass is not a test.

A second false signal in the same phase: a DHCP lease visible in the pfSense
table was treated as evidence that a client connection was healthy. A lease shows
an address was issued at some point. It does not show the connection is usable
now.

---

## Phase 7: The DNS failure

This is the most instructive problem in the build.

**Symptom:** Clients on the lab SSID received an address and a gateway, could
reach `8.8.8.8` by IP, and could not resolve any name.

**First hypothesis, wrong:** pfSense's Unbound resolver defaults to listening on
the LAN interface only, and LAN had been set to IPv4 `None`. That would explain
it exactly. Checked, and Unbound was set to listen on all interfaces.

**Second hypothesis, wrong:** DHCP was not handing out a DNS server. `ipconfig
/all` showed no DNS server line, which fit. Checked the DHCP configuration on the
lab interface and it was correctly set to hand out 192.168.30.1.

**Third hypothesis, correct:** The firewall rule order. The lab interface has a
block to `192.168.0.0/16` at the top. The resolver clients were being told to use
is 192.168.30.1, and that address sits inside `192.168.0.0/16`. The rule was
catching DNS queries to the gateway itself.

**Fix:** A pass rule above the block, protocol TCP/UDP, source the interface
subnet, destination the interface address, port 53. Applied to the IoT and Guest
interfaces as well, which had the same latent problem.

**Why it took three attempts:** Every wrong hypothesis was individually
reasonable and each one fit the symptom. The rule was also doing exactly what it
had been told to do, so nothing was broken in the sense of malfunctioning. This
is the fail-closed trade-off in one sentence: block all private space and you
will eventually block a service you depend on, because your own infrastructure
lives in private space too.

**Diagnostic note:** An active VPN tunnel on the test laptop was discovered
partway through this arc. A live tunnel routes DNS somewhere else entirely and
would have masked the real behaviour. Worth ruling out early in any name
resolution problem.

---

## Phase 8: Edge cutover

**Goal:** Remove the consumer router, eliminate double NAT, make pfSense the
only routing device.

Up to this point the old router stayed at the edge as a rollback path, with
pfSense's WAN sitting on its network at 192.168.1.250 and accepting double NAT.
That was deliberate. Removing the safety net before the replacement is proven
means a failure leaves the household without internet.

**Cabling decision:** pfSense's WAN had been reaching the old router through the
switch on VLAN 1. Two options existed at cutover: plug the modem into a switch
port on VLAN 1 and leave the WAN cable where it was, or run a cable directly from
the modem to ETH1 and keep the WAN physically off the switch fabric entirely.

The direct cable was chosen. It costs one cable run and removes any possibility
of a VLAN misconfiguration exposing the WAN to internal traffic. On a network
built to demonstrate segmentation, keeping untrusted traffic off the internal
switch fabric is the defensible answer.

**Two loose ends after the physical move:**

1. pfSense's WAN configuration was still statically pointed at the old router's
   network and had to be cleared and reconfigured for the modem.
2. The modem was provisioned to the old router's MAC address. The ISP had to
   re-provision it for the new device before the WAN came up.

**Outcome:** Single NAT, one routing device, consumer router removed from the
network entirely.
