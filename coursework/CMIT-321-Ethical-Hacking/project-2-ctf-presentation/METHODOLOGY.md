# Methodology - Two Challenges in Depth

## Category 10 Challenge 1: Admin Account Password (John the Ripper)

On the Category 10 Virtual Machines VM, the task was to locate the password hash for the admin account and crack it with John the Ripper, which the assignment said was already installed.

Combined the system's password and shadow files first with `unshadow /etc/passwd /etc/shadow > ~pwd.txt` - this merges the account list with the actual hashes into one file John can read, since the shadow file alone doesn't include full account rows.

The John the Ripper installed on the VM turned out to be version 1.6, an older release than the syntax shown in the class lab material, so the lab commands didn't transfer directly. An external wordlist attempt using modern long-option syntax (`--wordlist=`) failed with "Unknown option," because this version only accepts single-dash flags with the value attached directly to a colon (for example `-wordfile:FILE`). The wordlist file the lab referenced also couldn't be located on this system, so that approach was dropped.

Running `john` with no arguments prints the full usage text for the installed version, including the line `-users:[-]LOGIN|UID[,..]`. That notation was initially misread several times as literal characters to type, rather than placeholder notation meaning "supply either a login name or a UID here." Once trial and error stopped making progress, a screenshot of that manual line was run through Google Lens and asked to explain the syntax - it clarified that the value has to attach directly to the colon with no space, and that a UID works as well as a login name. Applying that, `john -users:1003 ~pwd.txt` loaded the target hash and cracked it almost immediately. Verified with `john -users:1003 -show ~pwd.txt`, which printed `administrator:P@ssw0rd:1003:1003::/home/administrator:/bin/sh` and confirmed one password cracked, zero left.

**Flag/Answer:** P@ssw0rd

## Category 6 Challenge 4: Admin's Password via FTP-Uploaded File

This challenge used a new packet capture file, so the same protocol assumptions from earlier challenges in the bank couldn't be carried over. Started the same way every new file gets approached: Statistics > Protocol Hierarchy with no filter, to get a full inventory of every protocol present before committing to one. Telnet showed up and was tried first since it had worked on a previous challenge, but that TCP stream turned out to be a short, unrelated fragment - a typo'd command and a disconnect, no login exchange at all.

The Follow Stream window shows a stream number in the corner, which was the clue that this capture held many separate TCP conversations, not just the one already followed. Went back to a plain `telnet` filter to see the full spread of Telnet traffic and stepped through stream numbers from 0 upward, finding nothing relevant until stream 62. That stream turned out to be FTP traffic - a different protocol than the one originally filtered for. It showed an anonymous FTP login followed by a `PORT` command and a `STOR` command uploading a file named `mypasswordfile.txt`.

The control channel only carries the FTP commands themselves - the same control-versus-data distinction encountered in an earlier FTP challenge in this bank. The actual file bytes travel over a separate data connection whose address is encoded directly in the `PORT` command line: `PORT 192,168,1,7,192,11`. FTP's active-mode PORT syntax packs the destination as six comma-separated numbers - the first four are the IP address, and the last two combine into the port number using `(first x 256) + second`. Working that out: `192 x 256 = 49152`, plus `11`, gives port 49163. Located the corresponding data-channel stream (stream 64) using that port number, followed it, and read the plaintext contents of `mypasswordfile.txt` directly.

**Flag/Answer:** UMGC-5545512, read straight from the uploaded file's contents in the reconstructed data stream.

**Reflection:** Going back through Protocol Hierarchy after the fact, FTP Data has its own nested "Line-based text data" entry with exactly one packet in it - the `STOR` upload itself. Drilling into that single node as a filter would have gone straight to the password without ever needing to decode the `PORT` command's IP/port math. The same lesson showed up again in Challenge 5 with TFTP's nested Data child: when a protocol's hierarchy entry has its own Data or text-data leaf with a small packet count, drill into that first before doing anything more manual.
