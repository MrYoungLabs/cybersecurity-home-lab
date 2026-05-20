# Tools and Resources — Project 1

A running list of tools, techniques, and reference sites used (or considered) while working through the CTF write-up.

## Reconnaissance and scanning

| Tool | Use |
| --- | --- |
| nmap | Port and service discovery |
| Nikto | Web server vulnerability scanning |
| WhatWeb | Web technology fingerprinting |
| Shodan | Internet-exposed device intelligence |

## Web exploitation

| Tool | Use |
| --- | --- |
| Burp Suite (Community) | Intercepting proxy, repeater, decoder |
| OWASP ZAP | Open-source alternative to Burp |
| sqlmap | Automated SQL injection probing |
| ffuf / gobuster | Content and parameter discovery |

## Forensics and crypto

| Tool | Use |
| --- | --- |
| Wireshark | Packet capture analysis |
| CyberChef | Encoding, decoding, hashing |
| exiftool | File metadata inspection |
| binwalk | Embedded file extraction |

## Password and credential work

| Tool | Use |
| --- | --- |
| John the Ripper | Offline password cracking |
| hashcat | GPU-accelerated cracking |
| hydra | Online credential brute force |
| seclists | Curated wordlists |

## Reference sites

- [HackTricks](https://book.hacktricks.xyz/) — broad attacker-perspective reference
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) — categorized payload library
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/) — defender-perspective reference
- [GTFOBins](https://gtfobins.github.io/) — Unix binaries for privilege escalation
- [LOLBAS](https://lolbas-project.github.io/) — Windows living-off-the-land binaries
- [CTF Field Guide](https://trailofbits.github.io/ctf/) — introductory CTF methodology

## Personal notes

- Add a one-line note each time a tool saves time on a challenge.
- Flag tools that didn't fit so you remember not to reach for them next time.
