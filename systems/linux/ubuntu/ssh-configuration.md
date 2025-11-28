# SSH Access & Security Configuration
A clean, predictable SSH baseline for homelab Ubuntu servers.
This builds directly on the Post-Installation Configuration guide and prepares your system for secure, reliable remote access.

The process is staged to avoid accidental lockouts:

1. Baseline SSH setup
2. Apply safe hardening
3. Enable and test key-based authentication
4. Disable password authentication
5. Add Fail2ban for local protection

---

## Before You Begin
You should have:

- completed the Ubuntu Server Post-Installation Configuration guide
- console or hypervisor access available (just in case you mis-type something)

---

# Phase 1 — Baseline SSH Setup

SSH is intentionally skipped during installation so you start with a controlled, predictable configuration.

This next part is a little tedious and takes a few minutes, but it’s absolutely worth doing properly. 
A lightly hardened SSH setup is miles better than blindly accepting the defaults — especially in a homelab where you want things secure, stable, and quiet.

This phase lays the foundation: a safe, functional SSH service that you can build on later.
If you ever want to extend things with extras like honeypots, port knocking, or more advanced intrusion detection, this baseline keeps you in a good place to do that.

## Install OpenSSH Server

```bash
sudo apt-get install openssh-server
```

---

## Configure SSH (Safe Baseline)

Open the SSH configuration file:

```bash
sudo nano /etc/ssh/sshd_config
```

Apply the following changes.

### Change SSH Port

```
# Port 22
```

→

```
Port 2022
```

### Disable remote root login

```
PermitRootLogin yes
```

→

```
PermitRootLogin no
```

### Limit authentication attempts

```
# MaxAuthTries 6
```

→

```
MaxAuthTries 3
```

### Reduce allowed sessions

```
# MaxSessions 10
```

→

```
MaxSessions 2
```

### Ensure public key auth is enabled

```
# PubkeyAuthentication yes
```

→

```
PubkeyAuthentication yes
```

### Disable X11 forwarding

```
# X11Forwarding yes
```

→

```
X11Forwarding no
```

### Disable SSH agent forwarding

```
# AllowAgentForwarding yes
```

→

```
AllowAgentForwarding no
```

### Idle timeout

```
# ClientAliveInterval 0
# ClientAliveCountMax 3
```

→

```
ClientAliveInterval 600
ClientAliveCountMax 2
```

### Increase SSH logging

```
# LogLevel INFO
```

→

```
LogLevel VERBOSE
```

Save and exit:

- Ctrl+O, Y, Ctrl+X

---

## Update UFW Rules

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2022/tcp comment "SSH"
sudo ufw enable
```

---

## Restart SSH

```bash
sudo systemctl restart ssh
```

Check the port:

```bash
sudo ss -tuln | grep 2022
```

If nothing appears, don’t panic — I’ve had this happen on fresh installs.
A quick reboot usually kicks systemd into applying the updated socket bindings properly:

```bash
sudo reboot
```
After the reboot, run the check again.

Before going any further, open another terminal or device and test logging in with your password to confirm the new port and baseline settings are working as expected.

*From another device on the same network:* `ssh -p 2022 youruser@server-ip-or-hostname`.

If login doesn’t work: stop here and troubleshoot.

---

# Phase 2 — Key-Based Authentication

Password-based logins are functional but weaker and less auditable.  
Key-based authentication is stronger, cleaner, and predictable.

This phase covers:

- generating SSH keys (Linux, macOS, Windows)
- installing your public key on the Ubuntu server
- confirming key-based access works

## Generate SSH Key — Linux / macOS

```bash
ssh-keygen -t ed25519 -C "yourname@hostname"
```

Accept defaults unless you have specific requirements.

---

## Generate SSH Key — Windows

```powershell
ssh-keygen -t ed25519 -C "yourname@hostname"
```

---

## Install Your Public Key on the Server


1. On your client display your public key:

   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```

2. Copy the full line starting with `ssh-ed25519`

3. On the server:

   ```bash
   mkdir -p ~/.ssh
   chmod 700 ~/.ssh
   nano ~/.ssh/authorized_keys
   ```

4. Paste the key
5. Save and exit
6. Set permissions:

   ```bash
   chmod 600 ~/.ssh/authorized_keys
   ```

---

## Test Key-Based Login

```bash
ssh -p 2022 youruser@server
```

You should log in using your key (and optionally your key passphrase), not your account password.

Do **not** disable password authentication until this works flawlessly. 

This really hurts when you're used to having copy/paste for commands and need to go back to manually typing them out!

---

# Phase 3 — Disable Password Authentication

Once keys work, lock things down.

Edit:

```bash
sudo nano /etc/ssh/sshd_config
```

Ensure:

```
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM yes
```

Restart:

```bash
sudo systemctl restart ssh
```

Test again from a **second terminal** before closing your existing session.

---

# Phase 4 — Fail2ban

Fail2ban prevents repeated login failures from cluttering logs or causing lockouts.

Install:

```bash
sudo apt-get install fail2ban
```

Create local config:

```bash
sudo nano /etc/fail2ban/jail.local
```

Add:

```
[sshd]
enabled = true
port = 2022
maxretry = 3
findtime = 10m
bantime = 30m
```

Restart:

```bash
sudo systemctl restart fail2ban
```

Check:

```bash
sudo fail2ban-client status sshd
```

---

# Why These Choices Were Made

You can configure SSH a dozen different ways, but this baseline aims for the sweet spot:
**secure enough that nothing silly gets through, predictable enough that future-you doesn’t curse past-you**, and **simple enough to repeat across every server without thinking**.

Each decision below supports that goal.

---

## Manual SSH Install
Letting the installer enable SSH sounds convenient, but it quietly accepts whatever defaults Ubuntu ships with — and those defaults change over time.

Installing SSH manually gives you:

- a **clean, intentional starting point**
- the **same baseline across every machine**, no surprises
- no hidden extras enabled without your knowledge

It keeps all servers consistent and avoids the classic “why is this option on?” mystery tour.

---

## Changing the SSH Port
Port 22 is scanned constantly. Even behind NAT, random bot traffic still shows up in logs.

Switching to a predictable but non-default port:

- **cuts log noise significantly**
- makes real anomalies stand out
- reduces background nonsense without pretending to be “security through obscurity”

It’s not about hiding SSH — just keeping the signal-to-noise ratio civilised, and remember, you can choose any number of ports to run SSH!

---

## No Root SSH Login
Direct root logins flatten audit trails. Everything becomes “root did it”, which is spectacularly unhelpful when debugging.

Disabling root logins:

- **forces named user accounts**, which improves traceability
- keeps privilege escalation explicit with `sudo`
- makes logs meaningful when reviewing actions later

When something changes, you want to know *who* changed it — not just that it was “root”.

---

## Authentication & Session Limits
Linux will happily accept multiple failed attempts and let you open a small forest of sessions if you let it.

Tightening these limits:

- stops brute-force attempts early
- encourages intentional logins
- prevents servers filling with forgotten shells from “that maintenance job I *definitely* finished”

Good hygiene supports good security — and vice-versa.

---

## Public Key Authentication
Passwords are convenient, but they carry all the usual risks: guessing, reuse, mishandling, or just being typed over someone’s shoulder.

SSH keys offer:

- **stronger cryptography**
- per-device revocation
- no password complexity hassles
- predictable, reproducible behaviour across clients

For almost zero effort, you gain a massive security upgrade.

---

## Disable X11 & Agent Forwarding
These features exist for niche workflows:

- X11 forwarding → running GUI apps remotely
- agent forwarding → letting remote hosts use your *local* SSH agent

Neither is useful on a homelab server, and both widen the attack surface.

Turning them off:

- removes unnecessary credential exposure paths
- avoids GUI-related complications
- keeps SSH focused on safe, simple shell access

If you ever genuinely need these, you’ll know — and you can re-enable them intentionally.

---

## Idle Timeouts
SSH sessions get abandoned. Tabs close. Wi-Fi drops. You pop away for five minutes, get distracted, and return tomorrow.

Idle timeouts:

- automatically clean up stale shells
- reduce the risk of someone stumbling into an unattended session
- keep your active connections honest

Ten minutes is a balanced default: long enough for normal tasks, short enough to avoid accidental all-day sessions.

---

## Verbose Logging
`LogLevel VERBOSE` isn’t noisy — it’s helpful.

Enabling it:

- records *what* authentication method was attempted
- highlights misconfigured keys or repeated failures
- integrates cleanly with Fail2ban’s pattern matching

It makes troubleshooting less guesswork and more clarity.

---

## Fail2ban
Even internal networks see repeated login failures from:

- old saved credentials
- typo storms
- over-enthusiastic scripts

Fail2ban:

- watches for repeated failures
- blocks offending IPs temporarily
- stops your logs turning into a wall of identical denial messages

It’s lightweight, predictable, and perfect for homelabs.

---

# Safety Notes

- Always test SSH changes in a **second terminal**
- Don’t close your working session until you’ve confirmed everything works
- Keep **hypervisor/console access** open when making SSH changes
- If you break SSH: revert the last change in `sshd_config`, restart the service, test again

Simple habits that prevent complicated evenings.
