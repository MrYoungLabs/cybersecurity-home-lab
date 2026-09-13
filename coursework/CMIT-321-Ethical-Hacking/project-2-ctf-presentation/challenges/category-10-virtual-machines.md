# Category 10 - Virtual Machines

Three challenges solved directly on a provided Linux VM (downloaded and run locally in VMware Workstation).

## Challenge 1 - Admin account password (John the Ripper)

**Flag/Answer:** P@ssw0rd

Full walkthrough in `METHODOLOGY.md` - this is one of the two challenges documented there in depth. Took roughly six hours spread across a day and a half, almost entirely because the VM's installed John the Ripper was an old version (1.6) whose command syntax didn't match the modern syntax taught in the class lab.

**Validated:** screenshots of the unshadow/merge step, the failed modern-syntax attempts, the John the Ripper usage/manual output, and the successful crack confirmed with `-show`.

## Challenge 2 - Yoda's password

**Flag/Answer:** wise (all lowercase)

Applied the syntax worked out in Challenge 1 (`john -users:<UID> ~pwd.txt`) to a different account, this time targeting yoda's UID directly. Nearly tripped over the fact that a passwd/shadow-style line carries two different ID numbers (UID and GID) that happen to match by coincidence for one account but not another.

**Validated:** confirmed via the course's answer checker, plus screenshots of locating yoda's UID and the successful crack.

## Challenge 3 - Full path to the ldd file

**Flag/Answer:** /usr/bin/ldd

The one that nearly got abandoned. Cycled through half-remembered commands (`find`, `grep`, `git`) without traction, then genuinely set it aside and moved on to another task for a while. Came back, browsed the filesystem manually with `ls` at the root and then `ls bin`, and seeing the actual list of command names (rather than trying to recall one cold) jogged the memory that `which` was the tool for locating an executable's path. `which ldd` printed the answer immediately.

**Validated:** screenshots of the manual-browsing dead ends and the successful `which ldd` output.

## Not attempted

No Category 10 challenges beyond these three were attempted in the time available.
