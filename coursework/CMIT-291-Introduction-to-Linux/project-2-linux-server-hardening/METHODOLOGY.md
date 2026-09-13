# Methodology

Environment: Ubuntu Server VM in VMware Workstation, on an isolated VMware NAT network (`192.168.88.0/24`), managed from a Windows host over SSH via PowerShell.

## Part 1 - Web Server (Nginx)

**Objective:** Install a web server and host a page that proves it's actually configured, not just running.

Nginx was chosen over Apache for its event-driven architecture - a small number of worker processes handle many connections through non-blocking I/O, rather than assigning a process or thread per connection the way Apache's traditional model does.

Steps:
1. `sudo apt update` - refreshed the package index before installing anything.
2. `sudo apt install nginx -y` - installed Nginx.
3. `sudo systemctl status nginx` - confirmed the service was `active (running)` and enabled to start on boot.
4. Replaced `/var/www/html/index.html` with a custom HTML page identifying the project, rather than leaving the default Nginx welcome page in place.
5. `hostname -I` - found the VM's NAT-assigned IP address.
6. From a browser on the Windows host, navigated to the VM's IP and confirmed the custom page loaded - proving the server was both running *and* actually configured.

**Verification:** Custom page rendered correctly in-browser from the host, not just a successful `systemctl status` check.

## Part 2 - SSH (Key-Based Auth, Password Auth Disabled)

**Objective:** Install SSH, switch to key-based authentication, and disable password login entirely.

Steps:
1. `dpkg -l | grep openssh-server` and `systemctl status ssh` - confirmed SSH was not yet installed.
2. `sudo apt install openssh-server -y` - installed it.
3. Initial status showed `ssh.service` as disabled/inactive, with `ssh.socket` handling on-demand activation - Ubuntu 24.04's default socket-activation model, which starts `sshd` on demand rather than running it continuously.
4. `sudo systemctl enable --now ssh` - converted it to a conventional always-on service; confirmed `active (running)` and enabled.
5. `ssh-keygen` (on the Windows host) - generated an `ed25519` key pair. The private key stays on the connecting machine (the host); only the public key goes to the server.
6. Copied the public key into the VM's `~/.ssh/authorized_keys` over an SSH session authenticated with the account password - the one and only time password auth was used.
7. `ssh ericmby@<VM_IP>` - confirmed key-based login worked, reaching a shell prompt with no password prompt at all.
8. Edited `/etc/ssh/sshd_config`: set `PasswordAuthentication no` and `PubkeyAuthentication yes`, then restarted the service.
9. `ssh -o PubkeyAuthentication=no ericmby@<VM_IP>` (from Windows) - confirmed the change took effect: the connection was rejected immediately with `Permission denied (publickey)` instead of prompting for a password.

**Verification:** A forced-rejection test (pubkey auth deliberately disabled client-side) proved password login was actually turned off, rather than trusting the config file at its word.

## Part 3 - Firewall (UFW) + Fail2ban

**Objective:** Open only the ports the server needs, and add a second, behavior-based layer to police the one port (22) that has to stay open to everyone.

### UFW

1. `sudo ufw status` - confirmed UFW was inactive by default, even though the package was already present.
2. Allowed the required ports *before* enabling the firewall, and in this order:
   - `sudo ufw allow 22/tcp`
   - `sudo ufw allow 80/tcp`
   - `sudo ufw allow 443/tcp`
3. `sudo ufw enable` - enabling before allowing SSH would have immediately cut off the very SSH session used to configure the server, since UFW defaults to denying all incoming traffic. UFW itself warned about this: *"Command may disrupt existing ssh connections. Proceed with operation (y|n)?"* - answered yes, since port 22 was already allowed.
4. `sudo ufw status verbose` - confirmed the firewall was active, default policy was deny incoming / allow outgoing, and all three ports were listed `ALLOW IN` for both IPv4 and IPv6.
5. Re-verified from the Windows host: `ssh ericmby@<VM_IP>` connected immediately with no password prompt, confirming the firewall change hadn't broken legitimate access.

### Fail2ban

1. `sudo apt install fail2ban -y` - installed it; `sudo systemctl status fail2ban` confirmed it started automatically and was enabled.
2. Created `/etc/fail2ban/jail.local` rather than editing `jail.conf` directly, since `jail.conf` is overwritten on package updates and `jail.local` is the supported way to persist custom settings on top of it.
3. Enabled the `sshd` jail with `maxretry = 5`, `findtime = 600`, `bantime = 600` - five failed login attempts from the same source within a 10-minute window triggers a 10-minute ban.
4. `sudo systemctl restart fail2ban` - applied the config.
5. `sudo fail2ban-client status` - confirmed the `sshd` jail was active.
6. `sudo fail2ban-client status sshd` - confirmed a clean baseline: 0 currently failed, 0 total failed, 0 banned IPs.

**UFW vs. fail2ban:** UFW is a static, port-based access control list - it answers whether a given port is open at all. Fail2ban is dynamic and behavior-based - it watches authentication logs over time and temporarily blocks a specific source IP once it shows a pattern of repeated failures. Port 22 has to stay open for legitimate SSH access, and that same openness is exactly what a brute-force attacker would try to exploit - fail2ban exists specifically to police that exposure rather than duplicate what UFW already does.

See `CASE_STUDY.md` for the live test that validated this configuration actually works.
