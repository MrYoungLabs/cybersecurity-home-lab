# Case Study - Validating Fail2ban with a Live Brute-Force Test

Configuring fail2ban and confirming its status is idle isn't the same as proving it works. This case study documents deliberately triggering it, rather than taking the configuration at its word.

## Baseline Log Review

Before generating any test traffic, two logs were reviewed to establish a normal baseline:

- `sudo tail -n 20 /var/log/auth.log` - recent authentication-related events: `sudo` usage, session opens/closes, and routine cron activity running every five minutes in the background. This established what normal noise looks like, to distinguish it from anything that would actually warrant attention.
- `/var/log/fail2ban.log` - fail2ban's own record of what it was watching, confirming the `sshd` jail was live before any test traffic existed.

## Attack Simulation

Rather than waiting for a real attack, one was generated deliberately: from a second PowerShell window on the Windows host, `ssh baduser@<VM_IP>` was run six times in a row against a nonexistent username.

Observed client-side behavior:

| Attempt | Result |
|---|---|
| 1st-5th | `Permission denied (publickey)` - returned immediately |
| 6th+ | `Connection timed out` - no response at all |

The switch from an immediate rejection to a silent timeout was the first sign the ban had taken effect: fail2ban's default action drops banned traffic silently rather than sending an active rejection, so the client just waits for a response that never comes.

## Collateral Effect: Self-Lockout

The ban also reset the already-authenticated SSH session in a separate window - fail2ban bans by *source IP address*, not by which individual login attempt succeeded or failed, so both the simulated attack and the legitimate working session were coming from the same address once that address was banned. This temporarily cut off all network access to the VM.

Recovery used the VMware console window instead of SSH, since local console login is independent of the network firewall and was never affected by the ban.

## Detection Evidence

Logged in directly at the console:

- `sudo fail2ban-client status sshd` → `Total failed: 6`, `Total banned: 1` - confirming the jail detected and acted on the attempt.
- `/var/log/fail2ban.log` showed the complete lifecycle:

| Time | Event |
|---|---|
| 18:45:51-18:46:01 | Five `Found <HOST_IP>` filter matches |
| 18:46:02 | `Ban <HOST_IP>` notice |
| 18:56:01 | `Unban <HOST_IP>` notice |

The unban fired exactly 599 seconds after the ban - matching the configured 600-second `bantime` almost to the second. This confirmed both that the jail was actively monitoring authentication attempts in real time, and that the temporary ban expired automatically as designed, with no manual intervention.

## Outcome

After the ban lifted, SSH access from the Windows host worked normally again - the full configure → verify → deliberately break it → confirm it recovers loop, closed out cleanly.
