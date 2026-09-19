# Security notes

Design rationale, the trade-offs that were made on purpose, and the gaps that
are still open. The open items are listed because a write-up that presents a
home network as finished and airtight is not credible.

## What this design is defending against

The realistic threat model for a household network, in rough order of likelihood:

1. **A compromised IoT device.** Embedded devices ship with firmware that is
   rarely patched and sometimes never updated. The assumption is that one of
   them will eventually be running something it should not be.
2. **A compromised household endpoint.** A laptop or phone that picks something
   up from a download or a malicious page.
3. **A guest device.** Not assumed hostile, but not managed either, and not
   something to place on the same segment as the infrastructure.
4. **The lab itself.** Security tooling and deliberately vulnerable targets live
   on VLAN 30. That segment is the most likely to be running something dangerous
   at any given moment, and it is treated as the least trusted network in the
   building.

What this does not defend against is a determined attacker with physical access,
or a compromise of pfSense itself. Both would be total.

## Core design decisions

### Segmentation is enforced at the routing layer, not the switch

The Catalyst is layer 2 only. It tags and forwards. Every policy decision about
what one segment may say to another is made by pfSense, in one place, in a rule
set that can be read top to bottom.

Splitting policy between a switch ACL and a firewall rule set means two places
to look when something is blocked and two places to forget when something needs
opening. One enforcement point is easier to reason about and easier to audit.

### Block the whole private range, not a single subnet

The lab's block rule covers `192.168.0.0/16` rather than naming one neighbouring
subnet. Any VLAN added in future is therefore denied by default and must be
explicitly permitted.

This is a deliberate choice about which way the failure points. A rule that names
one subnet fails open when the network grows, and it fails silently, which is
worse. A rule that covers the whole range fails closed and fails loudly, which is
how the DNS problem in the build log surfaced within hours rather than sitting
undiscovered.

### The management plane has no wireless presence

VLAN 40 carries Proxmox, the UniFi controller and switch management. There is no
SSID mapped to it and there never will be. Reaching the management network
requires a physical cable into a port configured for it.

This matters more than it first appears. The UniFi controller's administrator
password is recoverable by anyone with local access to its MongoDB store, which
was demonstrated during the build when credentials were lost. That is a normal
property of the software, not a defect, but it means the controller host has to
be treated as sensitive infrastructure rather than an appliance.

### The rollback path stayed until the replacement was proven

The consumer router remained at the edge through the entire build, with pfSense
behind it accepting double NAT. Double NAT is not good practice and it was not
the end state. It was a deliberate cost paid to keep a working network for the
household while the replacement was still unproven.

It came out only once every VLAN, every rule, every SSID and DNS resolution had
been verified by testing.

## Trade-offs accepted on purpose

**A lab SSID puts the isolated segment on the air.** VLAN 30 is reachable over
wireless from anywhere in the house. The alternative, keeping it wired only,
would have meant relocating to the room with the switch every time lab work
happened, which in practice means less lab work. Isolation is preserved because
pfSense still governs everything VLAN 30 can reach. The exposure added is that
the segment is joinable by anyone with the passphrase.

**Phone hopping instead of firewall holes for IoT.** Casting to a TV and
configuring smart devices needs discovery traffic, which is multicast and does
not cross a router. The options were an mDNS repeater plus permanent firewall
holes between the household and IoT segments, or joining the IoT SSID manually
for the few minutes those interactions take.

Hopping was chosen. It costs a manual wireless switch occasionally and it keeps
the isolation honest, with no standing permission between segments. A permanent
hole punched for convenience is permanent long after the reason for it is
forgotten.

**Phones live on the household VLAN, not IoT.** A phone is a general purpose
computer with a patch cycle and an app store. An IoT device is fixed function
embedded firmware nobody manages. They do not belong in the same trust zone
despite both being small and wireless.

**SSIDs are broadcast, not hidden.** Hiding an SSID is not a security control. A
client that has joined a hidden network broadcasts the name looking for it, so
the name is trivially recoverable, and hiding it degrades connection reliability
for legitimate devices. WPA3 with a strong passphrase is the actual control.

**The whole network depends on one host.** pfSense runs as a VM, so a Proxmox
failure takes down routing for the household. Start at boot on VM 100 handles the
reboot case. A hardware failure would not be handled, and the honest answer is
that this design trades availability for consolidation.

## Operational practices

- All switch configuration changes committed with `write memory` rather than left
  in running config
- The F4's port assignments physically labelled on the chassis, since interface
  names and silkscreen labels do not obviously correspond
- Firewall changes verified by testing from the affected segment rather than
  assumed from the rule table
- Saved wireless profiles removed from test clients before testing, so a client
  cannot silently fall back to another network and produce a false pass

## Open items

Tracked honestly rather than omitted.

| Item | Status |
|---|---|
| Switch management address still on the old network rather than VLAN 40 | Outstanding |
| UniFi controller on a 2 GB Pi 5, below the comfortable floor for MongoDB plus the Java application | Migration to a Proxmox VM planned |
| Cabling is temporary and untidy, correct lengths not yet run | Outstanding |
| Guest segment rules not yet audited as thoroughly as the lab segment | Outstanding |
| No IDS or IPS sensor on the trunk yet | Planned |
| No centralised logging or SIEM collection from pfSense and the switch | Planned |
| Vulnerability scanning of the lab segment with Nessus | Planned |
| Switch management over SSH with key authentication rather than the console | Planned |

## What this environment is for next

The segmentation work is infrastructure, not the destination. With isolated
segments and a firewall that logs, the next phase is putting the lab VLAN to use:
deliberately vulnerable targets on VLAN 30, scanning and exploitation from within
that segment, traffic capture at the trunk, and log collection into a SIEM so the
detection side gets exercised alongside the offensive side.

The value of doing segmentation first is that any of that work can now happen
without a mistake reaching the household network.
