# CMIT 291 - Introduction to Linux

Coursework folder for UMGC CMIT 291: Introduction to Linux. This is a working space for the graded hands-on Linux projects in the course - building a Linux VM from a clean install, then hardening it into a security-conscious server.

## What lives here

| Folder | Purpose |
|---|---|
| `project-1-vm-build-network-troubleshooting/` | Project 1 - Building and Configuring a Linux Environment |
| `project-2-linux-server-hardening/` | Project 2 - Configuring and Securing a Linux Server |

## Project list

**Project 1 - Building and Configuring a Linux Environment** (Due Week 3, individual)
Built an Ubuntu Server VM in VMware Workstation and configured the base environment. Mid-project, the VM lost network connectivity; root cause was traced to VMware's NAT networking service on the Windows host, not a guest OS misconfiguration. Restarting the service restored connectivity.

**Project 2 - Configuring and Securing a Linux Server** (Due Week 6, individual)
Installed and configured Nginx, set up SSH key-based authentication and disabled password login, configured a UFW firewall and a fail2ban sshd jail, then validated the setup with live log monitoring - including a deliberately triggered brute-force login test to confirm fail2ban actually detects and bans repeated failures (and auto-unbans once the ban window expires).

## Ground rules for this folder

- No UMGC assignment prompts or rubric text are reproduced verbatim; each write-up describes what was done, why, and what it verified, in the student's own words.
- - Screenshots are reviewed for anything sensitive before commit and are added by the student as a final step.
  - - IP addresses referenced are private, non-routable addresses on an isolated VMware NAT network created for these class VMs - not the student's home network.
    - 
