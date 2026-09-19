# Architecture

## Hardware

| Device | Role | Notes |
|---|---|---|
| CWWK F4 mini PC | Hypervisor and router | 16 GB DDR5 5600, 1 TB PCIe 4.0 NVMe, four Intel NICs |
| Cisco Catalyst 1000-16T | Layer 2 distribution | Hostname `Alpha-Lab-Switch-1`, 16 ports |
| Ubiquiti U7 | Wireless access point | WiFi 7, tri-band, 802.3bt PoE |
| 802.3bt PoE++ injector | Power for the AP | Active injector, not passive |
| Raspberry Pi 5 | UniFi Network Application host | 2 GB, wired to VLAN 40 |
| Cable modem | Edge | Provisioned by the ISP to the F4 MAC after cutover |

## Software

| Component | Version | Role |
|---|---|---|
| Proxmox VE | 9.2.2 | Type 1 hypervisor on bare metal |
| pfSense CE | 2.9.0-RELEASE | Routing, firewall, DHCP, DNS resolution |
| UniFi Network Application | 10.6.101 | AP management and SSID to VLAN mapping |

## Physical port map

The F4's four NICs are split between the host and the firewall VM. Passing all
four through to pfSense removes the host's own network path, so one is
deliberately held back.

| Port | Assignment | Notes |
|---|---|---|
| ETH0 | Proxmox host management | PCI device 02, retained by the host |
| ETH1 | pfSense WAN (`igc1`) | PCI passthrough, direct cable from the modem |
| ETH2 | pfSense trunk (`igc0`) | PCI passthrough, cable to switch Gi1/0/14 |
| ETH3 | Spare | Unused |

The port map is physically labelled on the chassis. Working out which silkscreen
label corresponds to which interface name cost real time during the build, and a
label removes that problem permanently.

## pfSense virtual machine

VM 100 on Proxmox:

- Machine type q35 with OVMF firmware, required for clean PCI passthrough
- 2 vCPU, 4 GB RAM, 20 GB disk
- Two NICs passed through via VT-d rather than bridged, so pfSense drives the
  hardware directly and the hypervisor is not in the packet path
- Start at boot enabled, so a host reboot brings the network back without
  intervention

### Interface configuration

`igc0` is set to IPv4 `None`. It carries no untagged address of its own and
exists purely as an 802.1Q trunk. Every VLAN is created as a child interface on
top of it and assigned individually.

| Interface | VLAN | Address | DHCP range |
|---|---|---|---|
| CYBERSECURITYLAB | 30 | 192.168.30.1/24 | .100 to .200 |
| MANAGEMENT | 40 | 192.168.40.1/24 | .100 to .200 |
| IOT | 50 | 192.168.50.1/24 | .100 to .200 |
| GUEST | 60 | 192.168.60.1/24 | .100 to .200 |
| MAINWIFI | 70 | 192.168.70.1/24 | .100 to .200 |

Static assignments on VLAN 40:

| Host | Address |
|---|---|
| pfSense gateway | 192.168.40.1 |
| Proxmox host | 192.168.40.10 |
| UniFi controller (Pi 5) | 192.168.40.101 (DHCP) |

## Switch configuration

The Catalyst does no routing. Its entire job is to tag frames, carry tagged
traffic on trunks, and drop untagged devices into the right VLAN on access
ports.

### Trunk to the firewall

```
interface gi1/0/14
 switchport mode trunk
 switchport trunk allowed vlan 30,40,50,60,70
```

### Trunk to the access point

The access point needs an untagged management address of its own, separate from
the tagged SSID traffic it carries. Untagged traffic on a trunk lands in the
port's native VLAN, which defaults to VLAN 1. VLAN 1 has no gateway and no DHCP
server in this design, so the AP would power up and never get an address.

Setting the native VLAN to 40 puts the AP's own management interface on the
management network while the SSIDs continue to ride tagged.

```
interface gi1/0/15
 switchport mode trunk
 switchport trunk native vlan 40
 switchport trunk allowed vlan 30,40,50,60,70
```

### Access ports

Unmanaged devices have no VLAN awareness. They send untagged frames and the
switch port decides where they land.

```
interface gi1/0/X
 switchport mode access
 switchport access vlan 50
```

The smart TV is wired this way, which puts it on the IoT segment with no
configuration on the device at all.

### Housekeeping

`no service config` was applied to stop the switch periodically attempting a
TFTP configuration fetch and logging errors when no server answered. All changes
are committed with `write memory`.

## Firewall policy model

Each VLAN interface has its own rule set. The ordering pattern on the segmented
interfaces is:

1. Pass TCP/UDP from the interface subnet to the interface address on port 53
2. Block from the interface subnet to `192.168.0.0/16`
3. Pass from the interface subnet to any

Rule two is the segmentation. Blocking all of `192.168.0.0/16` rather than a
single named subnet means every VLAN added later is denied by default, which is
the correct direction for the failure mode to point.

Rule one exists because rule two is thorough enough to break the network. The
gateway that serves DNS on each segment sits inside `192.168.0.0/16`, so the
private-space block catches DNS queries to the resolver. The narrow pass above
it restores name resolution without opening anything else. See
[BUILD-LOG.md](BUILD-LOG.md) for how that surfaced.

## Wireless

SSIDs are created in the UniFi controller and tagged with a VLAN ID. The
controller does not create networks on pfSense; pfSense already owns the subnets
and serves DHCP on them, so the controller's network objects are configured to
defer to a third party gateway.

| SSID | VLAN | Notes |
|---|---|---|
| CS | 30 | Lab access from anywhere in the house, so lab work is not tied to one room |
| Main | 70 | Household laptops and phones |
| Guest | 60 | Internet only |
| IoT | 50 | One wireless embedded device, plus deliberate phone hopping for casting |

All three radio bands are enabled on each SSID. 2.4 GHz matters in particular
for the IoT SSID, since a lot of embedded hardware cannot see a 5 GHz-only
network at all, and for the CS SSID, where 2.4 GHz provides the reach through
interior walls.

SSIDs are broadcast rather than hidden. A hidden network is not actually hidden,
because any client that has joined one broadcasts the name looking for it, and
hiding it degrades roaming for the devices that legitimately use it. WPA3 with a
strong passphrase does the real work.
