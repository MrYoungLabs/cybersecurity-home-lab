# Challenge Write-Up: Category 10 - Challenge 10

Course: CMIT 321 Ethical Hacking
Project: Project 1 - Capture the Flag Write-Up
Author: Eric Young
Status: Not completed during project window - documented for learning purposes

---

## Challenge Summary

**Category:** Category 10 - System Hacking
**Challenge:** On the provided VM, locate the password hash for the user account named admin. Use John the Ripper (already installed on the system) to crack the hash. Report the recovered password (case sensitive).
**Why not completed:** Time constraints. The Category 4 challenges were taking longer than anticipated, and I chose to stay focused on completing all 10 Category 4 challenges rather than starting a new category I could not finish.

---

## What the Challenge Asks

This challenge combines two distinct skills: locating a stored password hash on a Linux system, and using a password cracking tool to recover the plaintext password from that hash.

It is a foundational system hacking exercise that maps directly to real-world penetration testing tasks and to the credential-access phase of an attack (MITRE ATT&CK T1003: OS Credential Dumping).

---

## Why I Skipped It

By the time I had worked through the Category 4 challenges, I was behind my personal schedule and had limited time remaining. I had invested significant effort learning Splunk ingestion and SPL query patterns for Category 4 and wanted to ensure those 10 challenges were fully answered and documented.

Category 10 Challenge 10 requires working inside the UMGC-provided VM environment, which is a separate setup from the Splunk workflow I had built. Switching contexts and setting up the VM session would have taken time I did not have.

This is a challenge I intend to return to independently as part of building hands-on system hacking skills.

---

## How This Challenge Would Be Solved

### Step 1: Locate the password hash

On a Linux system, password hashes are stored in /etc/shadow. This file is only readable by root. To access it, the session must be running with root/sudo privileges.

```bash
sudo cat /etc/shadow | grep admin
```

This outputs a line in the format:

```
admin:$hash_type$salt$hash_value:...
```

The hash type prefix identifies the algorithm:
- $1$ - MD5
- $2a$ or $2y$ - bcrypt
- $5$ - SHA-256
- $6$ - SHA-512 (most common on modern Linux systems)

### Step 2: Save the hash to a file

John the Ripper operates on a file containing the hash. Extract the relevant line from /etc/shadow and save it:

```bash
sudo grep admin /etc/shadow > admin_hash.txt
```

Alternatively, use the unshadow utility (bundled with John) to merge /etc/passwd and /etc/shadow into a format John can parse directly:

```bash
sudo unshadow /etc/passwd /etc/shadow > combined.txt
grep admin combined.txt > admin_hash.txt
```

### Step 3: Run John the Ripper with a wordlist attack

John the Ripper supports several attack modes. For CTF challenges with weak passwords, a wordlist attack against the rockyou.txt wordlist is the standard first approach:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt admin_hash.txt
```

If rockyou.txt is not already decompressed:

```bash
gunzip /usr/share/wordlists/rockyou.txt.gz
john --wordlist=/usr/share/wordlists/rockyou.txt admin_hash.txt
```

### Step 4: View the cracked password

Once John finishes cracking, display the result:

```bash
john --show admin_hash.txt
```

This outputs the username and recovered plaintext password.

### Step 5: Note the exact case

The challenge specifies the answer is case sensitive. The password must be submitted exactly as John recovered it - do not lowercase or uppercase it.

---

## Alternative: hashcat

hashcat is another widely used password cracking tool, often faster than John for GPU-accelerated cracking. The equivalent hashcat command for a SHA-512 hash (mode 1800) with a wordlist attack:

```bash
hashcat -m 1800 admin_hash.txt /usr/share/wordlists/rockyou.txt
```

For this challenge, John the Ripper is specified as the required tool, but understanding hashcat is valuable for real-world engagements where GPU acceleration significantly reduces crack time.

---

## Key Concepts

### /etc/shadow and password storage

Modern Linux systems store password hashes in /etc/shadow (not /etc/passwd). The shadow file is only readable by root. During an authorized penetration test or forensic investigation, accessing shadow requires either root access or exploiting a privilege escalation vulnerability.

In the UMGC CTF VM, root access is provided as part of the lab environment. In a real penetration test, getting to /etc/shadow is a milestone that demonstrates successful privilege escalation.

### John the Ripper attack modes

John supports three main attack modes:

- **Wordlist mode:** Tries every word in a provided dictionary file. Fast and effective against common passwords.
- **Incremental (brute force) mode:** Tries all character combinations up to a specified length. Slow but exhaustive.
- **Rules mode:** Applies transformation rules to wordlist entries (e.g., append numbers, capitalize first letter). More effective than a straight wordlist against passwords like Password1.

For CTF challenges with intentionally weak admin passwords, a wordlist attack against rockyou.txt is almost always sufficient.

### Why password hashing matters

Passwords are stored as hashes rather than plaintext so that if the shadow file is exposed, an attacker cannot immediately use the passwords. A hash is a one-way mathematical function - given the hash, you cannot reverse it to the password directly. Instead, attackers use wordlist attacks or brute force to find a password that produces the same hash.

Weak or common passwords are crackable in seconds. Strong, unique passwords (long, random, not in any dictionary) are not practically crackable with current hardware. This is the argument for password managers and enforced complexity policies.

---

## SOC Relevance

This challenge maps to the credential-access and privilege escalation phases of an attack. In a SOC context:

- Detecting /etc/shadow access (auditd or EDR alerts on shadow file reads by non-root processes) is a high-confidence indicator of compromise
- John the Ripper and hashcat execution on a system should trigger alerts - they are not normal user applications
- Recovered credentials are often reused across systems (credential stuffing); a single cracked hash may unlock other accounts

Understanding how password cracking works makes the defensive controls - account lockout policies, long and random passwords, MFA - easier to justify and configure correctly.

---

## Tools That Would Be Used

| Tool | Purpose | Notes |
|---|---|---|
| /etc/shadow | Source of password hashes | Requires root access on Linux |
| unshadow | Merges /etc/passwd + /etc/shadow | Bundled with John the Ripper |
| John the Ripper | Password hash cracking | Pre-installed on UMGC lab VM |
| rockyou.txt | Wordlist for dictionary attack | Standard Kali/Parrot wordlist; 14+ million entries |
| hashcat | Alternative cracker (GPU-accelerated) | Not required for this challenge but useful context |

---

## References

[1] Openwall, "John the Ripper Password Cracker," Openwall, 2024. [Online]. Available: https://www.openwall.com/john/

[2] MITRE Corporation, "T1003: OS Credential Dumping," MITRE ATT&CK, 2024. [Online]. Available: https://attack.mitre.org/techniques/T1003/

[3] hashcat, "hashcat - Advanced Password Recovery," hashcat, 2024. [Online]. Available: https://hashcat.net/hashcat/

[4] The Linux Documentation Project, "Linux Password Security," TLDP, 2024. [Online]. Available: https://tldp.org/HOWTO/Shadow-Password-HOWTO.html

---

*No flag values, quiz answer keys, or copyrighted UMGC course content are stored in this file. This write-up documents method, tooling, and reasoning only.*
