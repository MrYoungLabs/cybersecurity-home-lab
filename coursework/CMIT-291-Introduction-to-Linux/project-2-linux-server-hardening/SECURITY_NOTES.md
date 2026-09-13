# Security Notes - Challenges, Lessons Learned, and What's Next

## Challenges Encountered

**Console screen blanking.** The VMware console window went black partway through the project. Cause: Linux text consoles blank themselves automatically after a period of inactivity, and since most of the work was happening over SSH, the console itself had gone untouched long enough to time out. Fix: clicking into the window and pressing a key woke it back up immediately - not a crash, just an idle timeout.

**Self-lockout during the fail2ban test.** Deliberately triggering fail2ban to test it locked out SSH entirely, since fail2ban bans by source IP rather than by which specific login attempt succeeded - and the legitimate working session was coming from the same IP as the simulated attack. Recovered using the VMware console directly, since local console login doesn't depend on the network at all and was never affected by the ban.

Both issues point at the same underlying lesson: **always have a way back into a system that doesn't rely on the thing you're actively testing or changing.**

## Lessons Learned

The biggest lesson was the value of a **configure → verify → test** loop, rather than assuming a change worked just because the command ran without an error. Almost every step in this project followed that pattern:

- Configure UFW → check its status → re-test SSH access
- Configure fail2ban → deliberately trigger it → confirm it actually banned, then later auto-unbanned as designed
- Disable password authentication → prove it with a forced-rejection test, rather than taking the config file at its word

This maps directly onto the CompTIA troubleshooting methodology's final step: verifying full functionality with the user before closing out a ticket. Carrying that same discipline into future work - treating a configuration change as unverified until it's actually been tested, not "finished the moment it's applied" - is the main takeaway from this project.

This project also built real confidence in securing a Linux server hands-on, and it directly motivated applying the same UFW + fail2ban approach to a home network setup, not just a disposable class VM.

## What's Next

Given more time, the SSH brute-force scenario would be worth re-running from a genuinely separate device (a phone) instead of a second window on the same host, for a more realistic out-of-band test. That would require configuring port forwarding on VMware's NAT network first, since a phone on the same Wi-Fi can't reach the VM's private NAT subnet without it - a good follow-up exercise in understanding NAT boundaries and out-of-network attack surface testing.
