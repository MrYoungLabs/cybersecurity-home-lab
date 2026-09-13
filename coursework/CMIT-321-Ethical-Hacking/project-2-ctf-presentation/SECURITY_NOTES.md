# Lessons Learned

## Strengths / how these skills would benefit a CTF team

The greatest strength coming out of this project is persistence - sticking with a problem instead of giving up after the first, second, or third approach fails. In practice that shows up as a systematic, one-hypothesis-at-a-time approach: try something, rule it out cleanly, then move to the next idea, rather than clicking through everything hoping something looks interesting. In Category 6 Challenge 5, Telnet, POP3, a service mislabeled as Kpasswd, a full Nmap SMB scan, and anonymous FTP probing were all worked through before the real answer turned up in TFTP - reached only by generalizing from a technique that had already worked (FTP handing over a password file in Challenge 4) to a different but related protocol. Reconnaissance activity hiding behind a mismatched port also showed up twice (Challenge 5's SMB-on-the-Kpasswd-port, Challenge 6's Rlogin-that-was-actually-a-scanner), which reinforced reading the raw bytes instead of trusting a port number to tell the whole story. A habit of verifying answers a second way also held up: Challenge 1 was re-solved from scratch through a completely different path just to confirm the first answer wasn't a fluke.

## Easiest challenge banks

Category 6 Challenges 1 through 3 were the most straightforward once the core technique was in place: filter by protocol name, then read credentials straight out of an unencrypted protocol - SSH's handshake direction field, FTP's plaintext `PASS` command, or a reconstructed Telnet stream. Challenge 7 is a good example of the payoff from understanding Wireshark's filter syntax: knowing to filter on `ftp.response.code == 230` went straight to the one successful login instead of scrolling past tens of thousands of failed dictionary-attack attempts. Category 10 Challenge 2 also became easy, but only after Challenge 1 taught the `-users:` syntax needed to target one specific account by UID.

## Areas needing more practice

Baseline Linux command-line fluency is the weakest area, not the security concepts themselves. Category 10 Challenge 3 - which only asked for the full path to the `ldd` file - took far longer than it should have, cycling through `find`, `grep`, and even `git` before finally just browsing `/bin` and recognizing `which` on sight rather than recalling it cold. Reading unfamiliar tool documentation under pressure is a related gap: the John the Ripper manual line `-users:[-]LOGIN|UID[,..]` cost hours because it initially read as literal characters to type rather than placeholder notation.

## Where the most struggle happened

Category 10 was the hardest overall, especially Challenge 1, which took roughly six hours spread across a day and a half - almost entirely because the VM's John the Ripper was an old version (1.6) whose syntax didn't match what the class lab taught. The single most costly moment was misreading that manual line as literal syntax rather than notation; that misinterpretation, more than the version mismatch itself, is what caused the stall until Google Lens helped break it down. Category 6 Challenge 5 and Category 10 Challenges 1 and 3 were the closest calls to giving up partway through before working through to an answer.

## Not completed

Within Category 6, the "find the admin password" challenge bank runs at least through Challenge 10; only Challenges 1 through 7 were completed. Challenges 8, 9, and 10 in that bank were not attempted, mainly due to time rather than running out of ideas. Everything else attempted across both categories was completed and verified.

## How to improve

For command-line fluency: more hands-on repetition outside of a guided lab - working through unguided Linux challenges on a platform like Hack The Box so commands like `which`, `find`, and `grep` become instinct rather than something to reconstruct under pressure. For reading unfamiliar tool documentation: build the habit of running a tool with no arguments or `--help` before trying anything else, rather than guessing at syntax first and only reaching for outside help after getting stuck. A personal reference of older-tool syntax quirks (like John 1.6's colon-attached flags) would also help avoid rediscovering the same thing from scratch on the next unfamiliar system.
