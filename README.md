> **Update:** The current, most advanced build in this repo is the Proxmox + pfSense + Cisco Catalyst segmented network and security lab, documented in full in [network-segmentation-build/](network-segmentation-build/) (README, architecture, build log, and security notes). The write-up below documents an earlier double-NAT / antiX Linux iteration of this project.
---
# Cybersecurity Home Lab Project

**Project Start Date:** April 2026  
**Certifications:** CompTIA Network+ CE, CompTIA Security+ CE, AWS Cloud Practitioner, AWS Solutions Architect Associate  
**Goal:** Build a hands-on home lab environment to develop practical cybersecurity skills and support career development toward an IT Help Desk role and beyond.

---

## Hardware Overview

| Role | Category | Purpose |
|---|---|---|
| Primary Laptop | Modern laptop, Windows 11 | Main workstation, research, Hack the Box |
| Lab Machine | Legacy desktop, repurposed | Linux lab server, network monitoring |
| Primary Router | Wi-Fi 6 capable home router | ISP gateway — hosts home & IoT networks |
| Secondary Router | Business-grade wired router | Office router (double NAT) — hosts lab & bedroom IoT networks |

> Specific hardware models and specifications are intentionally omitted from this documentation.

---

## Project Overview

This project documents the build-out of a personal cybersecurity home lab using repurposed hardware. The lab is designed to provide hands-on experience in Linux administration, network monitoring, and security tooling to complement existing industry certifications.

The project is structured in three phases:

- Linux installation and configuration on legacy hardware
- Network monitoring and server setup
- Active security practice and skill building

---

## Network Architecture

### Design Rationale

The home lab uses a **double NAT** topology with two routers and multiple VLANs. This architecture was not chosen arbitrarily — it emerged from three concrete requirements that shaped every design decision.

**Physical Connectivity Constraint:** The living room and office are separated by a structural wall that significantly degrades Wi-Fi signal quality. Running a single router across both rooms was not viable. The solution was to place the Primary Router in the living room (the ISP gateway) and connect the Secondary Router in the office via a wired WAN uplink. This physical necessity creates a natural double NAT boundary, which turned out to be a security advantage as much as a connectivity solution.

**Security Segmentation:** Cybersecurity lab work requires strict isolation from household devices. Running attack tools, vulnerable virtual machines, and network scanners on the same broadcast domain as general household devices or smart home kit presents unnecessary risk — both to household devices and to the integrity of lab experiments. The double NAT boundary means lab traffic is separated from the household network at the network layer, not just at the VLAN level. Even a misconfigured VLAN rule on the Secondary Router cannot expose lab traffic to the home network, because the NAT boundary at the Primary Router acts as a second independent barrier.

**Household Usability:** The primary household user needs a simple, stable network that is completely unaffected by lab activity. Placing the Primary Router as the household gateway ensures all household traffic routes through a clean, unmodified path to the ISP. Lab activity on the Secondary Router is entirely invisible to it.

---

### Double NAT Topology — Visual Diagram

```
                             INTERNET
                                 |
                           [ ISP Modem ]
                                 |
                    ┌────────────────────────┐
                    │     PRIMARY ROUTER     │   ← WAN: Public IP (from ISP)
                    │      (Living Room)     │     LAN: Home address space
                    │                        │
                    │  Segment A ───────────────── Home Network
                    │  Segment B ───────────────── Home IoT Network
                    │                        │
                    └──────────┬─────────────┘
                               │
                       Wired WAN Uplink
                  (Primary Router LAN port →
                   Secondary Router WAN port)
                               │
                    ┌────────────────────────┐
                    │    SECONDARY ROUTER    │   ← WAN: Address leased from Primary Router
                    │        (Office)        │     LAN: Separate address space
                    │                        │
                    │  Segment C ───────────────── Cybersecurity Lab Network
                    │  Segment D ───────────────── Bedroom IoT Network
                    │                        │
                    └────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │  KEY: Two NAT boundaries exist in this topology.               │
  │  Traffic originating in the Lab Network (Segment C) must       │
  │  traverse NAT at the Secondary Router, then NAT again at       │
  │  the Primary Router before reaching the internet.              │
  │  Lab devices have no initiated-path to the home network        │
  │  segments (A/B) — the Primary Router's NAT boundary blocks     │
  │  all unsolicited inbound traffic from the office by design.    │
  └────────────────────────────────────────────────────────────────┘
```

---

### Logical Network Segments

The network is divided into four logical segments across two routers. Rather than documenting specific subnet addresses, the design intent for each segment is described below.

| Router | Segment | Network Name | Purpose |
|---|---|---|---|
| Primary Router | A | Home Network | General household use — shared devices, primary household user |
| Primary Router | B | Home IoT | Smart home devices in common areas — isolated from household computers |
| Secondary Router | C | Cybersecurity Lab | Lab machines, VMs, attack/defence practice, network monitoring |
| Secondary Router | D | Bedroom IoT | Smart devices in the bedroom — isolated from lab traffic |

The two routers intentionally use **distinct, non-overlapping private address spaces**. This makes it immediately obvious in logs, packet captures, and monitoring output which router and segment a packet originates from — without needing to consult documentation. This is a deliberate operational choice to reduce cognitive overhead during active lab sessions.

#### Why Two Separate IoT Segments?

Smart home devices are physically distributed across rooms. Rather than consolidating all IoT onto a single VLAN, each router manages the IoT devices in its physical area. This approach:

- Eliminates cross-router IoT traffic for devices that only need local connectivity
- Ensures IoT devices remain isolated regardless of which router they happen to connect through
- Limits the blast radius of a compromised smart device — it can only reach other IoT devices on its own segment, not household computers or lab machines

---

### Logical Device Placement

Devices are assigned to segments based on their trust level and function, not their physical location alone.

**Home Network (Segment A — Primary Router)**  
General-purpose household devices. Any device used by household members for everyday tasks lives here. The Secondary Router's WAN uplink is also assigned an address in this segment — from the Primary Router's perspective, the entire office network is simply one more client device.

**Home IoT (Segment B — Primary Router)**  
Smart home devices in common areas: lights, speakers, displays, and similar kit. These devices are intentionally isolated from the home network — they can reach the internet, but cannot initiate connections to household computers or each other across segment boundaries.

**Cybersecurity Lab (Segment C — Secondary Router)**  
The primary workstation and lab machine both reside here. This is the most controlled segment in the network. Traffic from this segment passes through two NAT layers before reaching the internet, and has no routed path to either the home network or IoT segments. Future virtual machines for attack/defence practice will also be assigned addresses in this segment.

**Bedroom IoT (Segment D — Secondary Router)**  
Smart devices physically located in the bedroom, managed by the Secondary Router. Despite sharing a router with the lab, this segment is isolated from the lab by VLAN segmentation. Smart TV traffic, for example, cannot reach lab machines even though they share the same physical router.

---

### Security Properties of the Double NAT Design

This topology provides layered isolation that goes beyond what a single-router VLAN setup can offer.

- **Lab-to-home isolation:** Devices in Segments C and D cannot initiate connections to Segments A or B. The Primary Router's NAT boundary blocks all unsolicited inbound traffic originating from the office network. No port forwarding rules are configured to bridge this boundary.

- **Home-to-lab isolation:** Devices in Segment A have no routing knowledge of the Secondary Router's internal segments. The Secondary Router appears as a single client device to the Primary Router — its internal network topology is entirely opaque.

- **IoT containment:** Both IoT segments (B and D) are isolated from all other segments by VLAN enforcement. A compromised smart device cannot communicate with lab machines, household computers, or IoT devices on the other router.

- **Double NAT obfuscation:** Outbound lab traffic is translated twice before reaching the internet — once at the Secondary Router (replacing lab addresses with an address in the Primary Router's space), and again at the Primary Router (replacing that address with the ISP-assigned public IP). This provides an additional layer of address obfuscation for outbound lab activity.

- **Defence-in-depth:** The double NAT design means two independent security boundaries must both fail before lab traffic could reach household devices. A single misconfigured VLAN rule, firewall exception, or routing table entry is insufficient to bridge the isolation gap.

---

### Known Trade-offs & Future Improvements

Every design involves trade-offs. The following limitations are accepted consciously and documented for future resolution.

- **No IDS/IPS at the NAT boundary** — traffic crossing between the two routers is not currently inspected. A future improvement is to deploy an inline intrusion detection sensor (e.g., Snort or Suricata) on the lab machine to monitor Segment C traffic.

- **No inter-VLAN boundary logging** — traffic that attempts to cross segment boundaries is silently dropped rather than logged. Adding logging to boundary-crossing attempts is planned for Phase 2.

- **Double NAT complicates inbound connections** — any use case requiring inbound connectivity (e.g., receiving reverse shells in CTF exercises) requires port forwarding rules on both routers in sequence. This is an acceptable operational complexity given the isolation benefits.

- **Secondary Router WAN sits in Segment A** — the Secondary Router's WAN-facing address lives in the same segment as household devices on the Primary Router. A dedicated transit segment between the two routers would be architecturally cleaner but is not required at current scale.

---

## Phase 1 — Linux Installation on Lab Machine ✅ COMPLETED

### Objective

Install a secure, actively maintained Linux distribution on the legacy lab machine to repurpose it as a Linux server and network monitoring node.

### OS Selection Process

The lab machine uses a 32-bit processor architecture — a significant constraint that eliminated most modern Linux distributions from consideration.

**Option 1 — Debian:** Debian was the first choice due to its reputation as one of the most secure and stable Linux distributions with excellent long-term support. However, Debian dropped official 32-bit (i386) support after Debian 10 (Buster), making newer versions incompatible with the lab machine's hardware. Eliminated due to architecture incompatibility.

**Option 2 — Devuan:** Devuan is a Debian fork that removes systemd and was considered as an alternative with potentially better support for older hardware. However, it also presented compatibility issues with the lab machine's specific hardware configuration. Eliminated due to hardware compatibility issues.

**Option 3 — antiX Linux (Selected):** antiX Linux was chosen as the final option for the following reasons:

- Built specifically for older 32-bit hardware
- Actively maintained with ongoing security updates and patches
- Based on Debian, inheriting much of its security foundation
- Lightweight enough to run efficiently on aging hardware

> **Key Lesson:** A theoretically more secure OS is irrelevant if it doesn't support your hardware or is no longer receiving updates. Compatibility and active maintenance are critical selection criteria.

### Tools Used

| Tool | Purpose | Platform |
|---|---|---|
| Rufus | Create bootable USB drive from ISO | Primary Laptop (Windows 11) |
| antiX Linux 5.10.240-antix.1-486-smp | Linux operating system | Lab Machine |
| Windows PowerShell | Hash verification of downloaded ISO | Primary Laptop (Windows 11) |

### Step-by-Step Process

**Step 1 — Download antiX Linux ISO on Primary Laptop**  
Downloaded the antiX Linux ISO file from the official antiX website to the primary laptop.

**Step 2 — Verify ISO Integrity via Hash Check**  
Before proceeding with installation, the integrity of the downloaded ISO was verified using Windows PowerShell to ensure the file had not been corrupted or tampered with during download. This is a critical security practice.

Command used in PowerShell:
```powershell
Get-FileHash C:\path\to\antix.iso -Algorithm SHA256
```
The output hash was compared against the SHA256 hash published on the official antiX download page. They matched, confirming the file was legitimate and unmodified.

**Step 3 — Create Bootable USB Drive**  
Used Rufus on the primary laptop to write the verified antiX Linux ISO to a USB flash drive, creating a bootable installation media.

**Step 4 — Boot Lab Machine from USB**  
Inserted the bootable USB into the lab machine and configured it to boot from the USB drive. Successfully booted into the antiX Linux live environment.

**Step 5 — Install antiX Linux to Internal Hard Drive**  
Ran the antiX installer from the live environment, installing the OS to the internal hard drive. During setup, created a regular user account and set a root password. The installer confirmed the OS was installed to:

- Device: `/dev/sda`
- Partition: `/dev/sda1`

**Step 6 — Verify Installation**  
Before rebooting, opened a terminal from the live USB environment and verified the installation using the `lsblk` command, which confirmed antiX was correctly installed and mounted to the appropriate partition.

**Step 7 — Remove USB and Reboot**  
Removed the USB flash drive as instructed and attempted to reboot into the newly installed antiX system.

### Problems Encountered & Solutions

**Problem 1 — System dropped to `grub>` prompt on reboot**

*Symptom:* After removing the USB and rebooting, the system dropped to a `grub>` command prompt instead of booting into antiX

*GRUB Version:* GNU GRUB version 2.12-9+deb13u1

*Cause:* GRUB bootloader loaded successfully but could not locate the Linux kernel automatically

*Diagnosis:* Used `ls` command at the `grub>` prompt to confirm GRUB could see the drive: output showed `(hd0) (hd0,msdos1)`. Used `ls (hd0,msdos1)/` to confirm the antiX filesystem was present

*Solution:* Manually booted the system using the following GRUB commands

```
set root=(hd0,msdos1)
linux (hd0,msdos1)/boot/vmlinuz-5.10.240-antix.1-486-smp root=/dev/sda1 ro
initrd (hd0,msdos1)/boot/initrd.img-5.10.240-antix.1-486-smp
boot
```

**Problem 2 — Kernel filename not found at default path**

*Symptom:* Initial boot command using `/boot/vmlinuz` returned a file not found error

*Cause:* The kernel filename included the full version string rather than a generic name

*Solution:* Used `ls (hd0,msdos1)/boot/` to list exact filenames. Identified correct filenames: `vmlinuz-5.10.240-antix.1-486-smp` and `initrd.img-5.10.240-antix.1-486-smp`

**Problem 3 — Root login forbidden at console**

*Symptom:* Attempting to log in as root returned "root account login forbidden"

*Cause:* antiX Linux disables direct root login at the console by default as a security measure

*Solution:* Logged in using the regular user account created during installation, then used `sudo` for commands requiring elevated privileges

### Permanent GRUB Fix

After successfully booting into antiX, the GRUB configuration was permanently fixed to prevent needing manual boot commands in the future. Opened a terminal from the antiX desktop and ran:

```bash
sudo update-grub
```

GRUB scanned for operating systems, generated a new configuration file, and added a boot menu entry. On the next reboot, the system booted automatically into antiX without any manual intervention.

**Outcome:** antiX Linux fully installed, verified, and booting correctly and independently. ✅

### Key Lessons Learned from Phase 1

- Always document as you go. A session interruption without documentation means starting from scratch and losing hours of work.
- Hash verification of downloaded files is a simple but essential security practice that should never be skipped.
- GRUB issues on older hardware are common but fixable directly from the `grub>` prompt.
- Linux root login is typically disabled by default — this is a security feature, not a bug.
- Architecture compatibility (32-bit vs 64-bit) is a critical factor when selecting software for older hardware.
- antiX Linux is an excellent, actively maintained choice for legacy 32-bit machines.

---

## Phase 2 — Lab Configuration 🔄 IN PROGRESS

### Objective

Configure the lab machine running antiX Linux as a network monitoring node and/or server within the home lab environment.

Planned steps:
- [ ] Connect lab machine to the Cybersecurity Lab segment on the Secondary Router
- [ ] Assign a static address within the Lab Network segment
- [ ] Assess network monitoring tools compatible with antiX (e.g., Wireshark, ntopng, Snort, Suricata)
- [ ] Install and configure chosen monitoring tools
- [ ] Define and document the lab machine's specific role in the network

---

## Phase 3 — Security Practice & Skill Building 📋 PLANNED

### Objective

Use the home lab environment for active, hands-on cybersecurity practice to build skills beyond certification knowledge.

Planned steps:
- [ ] Set up Hack the Box on the primary laptop via browser
- [ ] Use the lab machine for network traffic analysis during practice sessions
- [ ] Document all exercises, findings, and lessons learned
- [ ] Build toward more advanced certifications and practical experience

---

## Resources & References

- antiX Linux: https://antixlinux.com
- Rufus: https://rufus.ie
- Hack the Box: https://hackthebox.com
- CompTIA: https://comptia.org
- AWS Certification: https://aws.amazon.com/certification

---

## About This Project

This home lab is being built and documented as part of an active journey into the cybersecurity field. All work is conducted in a personal, isolated lab environment for educational purposes.

---

*Documentation last updated: April 2026*
