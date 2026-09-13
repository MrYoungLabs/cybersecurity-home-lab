# Project 2 - Configuring and Securing a Linux Server

**Course:** CMIT 291 Introduction to Linux
**Due:** Week 6 (individual)
**Status:** Complete - submitted

## Assignment Summary

Configure a Linux server with a working web server, key-based SSH access with password authentication disabled, a host-based firewall (UFW), and fail2ban-based intrusion prevention - then verify each control with log monitoring, including deliberately triggering fail2ban to confirm it actually detects and bans brute-force login attempts (and auto-unbans once the ban window expires).

## Deliverables

| Deliverable | Status |
|---|---|
| Nginx web server with a custom hosted page | Complete - see `METHODOLOGY.md`, Part 1 |
| SSH key-based auth, password auth disabled | Complete - see `METHODOLOGY.md`, Part 2 |
| UFW firewall (22/80/443) + fail2ban sshd jail | Complete - see `METHODOLOGY.md`, Part 3 |
| Live log monitoring + fail2ban ban/unban test | Complete - see `CASE_STUDY.md` |
| Security notes and lessons learned | Complete - see `SECURITY_NOTES.md` |
| Tools and references | Complete - see `tools-and-resources.md` |
| Screenshots (`assets/`) | Pending - to be supplied by student |

## Folder Contents

| File / Folder | Purpose |
|---|---|
| `METHODOLOGY.md` | Step-by-step configuration: Nginx, SSH hardening, UFW, fail2ban |
| `CASE_STUDY.md` | Live attack simulation against fail2ban, with log evidence |
| `SECURITY_NOTES.md` | Challenges, lessons learned, and the UFW-vs-fail2ban distinction |
| `tools-and-resources.md` | Tools used and documentation referenced |
| `assets/` | Screenshots - pending, to be supplied by student |

## Notes on Integrity

No UMGC assignment prompts or rubric text are reproduced verbatim here. This documents configuration method, verification steps, and lessons learned in the student's own words.
