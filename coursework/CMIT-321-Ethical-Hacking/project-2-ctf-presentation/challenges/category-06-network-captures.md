# Category 6 - Network Captures/Wireless

Seven challenges solved by analyzing Wireshark packet captures. Each capture file is a separate scenario, so the working protocol/technique from one challenge was never assumed to carry over to the next without first re-checking Statistics > Protocol Hierarchy.

## Challenge 1 - SSH server IP address

**Flag/Answer:** 192.168.1.200

Filtered the capture on `ssh` to isolate the SSH handshake, then used the packet detail pane's `[Direction: Client to Server]` field on the first packet to identify which side of the connection was the server, and confirmed it against the packet's Destination IP field. Re-solved from scratch through a different path (Protocol Hierarchy first, `ssh` filter from there) as a self-check, and got the same answer.

**Validated:** screenshots of the Protocol Hierarchy SSH node and the packet detail confirming the destination IP.

## Challenge 2 - FTP password containing "UMGC"

**Flag/Answer:** UMGC-234562

Filtered on `ftp` and read the login exchange directly - since FTP is unencrypted, the `PASS` command carries the password in plaintext. This capture had more than one FTP session in it, including an unrelated anonymous login with no password of interest, so the login exchange had to be found by scrolling past that decoy rather than trusting the first FTP session found.

**Validated:** screenshots of the Protocol Hierarchy FTP branch and the packet detail showing `PASS UMGC-234562` in plaintext.

## Challenge 3 - Admin's password (Telnet capture)

**Flag/Answer:** P@ssw0rd

Statistics > Protocol Hierarchy with no filter gave a full inventory of the capture's protocols. Telnet stood out as the older, insecure predecessor to SSH. Right-clicked a Telnet packet and used Follow TCP Stream to reassemble the full interactive session into one readable transcript, which contained the admin password in plaintext.

**Validated:** screenshot of the Follow TCP Stream window (stream 1115) showing the login and password.

## Challenge 4 - Admin's password (FTP-uploaded password file)

**Flag/Answer:** UMGC-5545512

Full walkthrough in `METHODOLOGY.md` - this is one of the two challenges documented there in depth.

**Validated:** screenshots of the Protocol Hierarchy FTP branch, the control-channel `PORT`/`STOR` commands, the port-number calculation, and the Follow TCP Stream window on the matching data channel showing the file's plaintext contents.

## Challenge 5 - Admin's password (multi-protocol elimination)

**Flag/Answer:** UMGC-8887771

The hardest elimination process in this category. Telnet, POP3, a service running on the Kpasswd port that turned out to actually be SMB (mislabeled by Wireshark based on port number, not content), a full Nmap SMB scan, and anonymous FTP probing were all dead ends or automated-tool noise before TFTP - a protocol with no authentication step at all - handed over a file named `thep@ss.txt` in plaintext.

**Validated:** screenshots of the Protocol Hierarchy TFTP branch (with its nested Data child), the Read Request for the file, and the Follow UDP Stream showing the plaintext password.

## Challenge 6 - Password for a newly-created "superman" account

**Flag/Answer:** UMGC-2024321

Checked LDAP first (account provisioning association) but found only 4 packets, a genuine dead end. Checked Rlogin next but the traffic turned out to be an unrelated port scanner, not real rlogin sessions. Went back to Telnet (151 packets, real traffic this time), followed the stream, and found a full Microsoft Telnet login sequence followed by the literal Windows command `net user superman UMGC-2024321 /add` typed at the shell - the password sitting in plaintext as a command argument.

**Validated:** screenshots of the Protocol Hierarchy Telnet branch and the Follow TCP Stream window showing the `net user` command.

## Challenge 7 - Admin password behind an FTP dictionary attack

**Flag/Answer:** ended

The capture showed a long, alphabetically-ordered sequence of `USER admin` / `PASS <word>` attempts, each rejected - the signature of an automated dictionary attack rather than a human guessing. With over 336,000 packets in the file, scrolling through every failure wasn't practical. Filtered instead on `ftp.response.code == 230` (FTP's "user logged in" success code) to jump straight to the one successful attempt.

**Validated:** screenshot of the packet list showing the successful `PASS ended` / `230` login response.

## Not attempted

Category 6's "find the admin password" challenge bank runs at least through Challenge 10; only Challenges 1-7 were attempted and completed due to time. Challenges 8-10 were not attempted.
