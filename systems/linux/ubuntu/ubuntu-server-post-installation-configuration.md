# Ubuntu Server Post-Installation Configuration
A clean, predictable baseline for homelab Ubuntu servers. 

This guide applies a clean, minimal post-install baseline for Ubuntu Server, focused on security, predictability, and sustainability — with enough hardening to avoid silly problems without turning your homelab into a CIS-Level-4000 compliance nightmare.

## Repository Location

```
systems/
└── linux/
    └── ubuntu/
        └── ubuntu-ssh-configuration.md   
        └── ubuntu-server-installation.md        
        └── ubuntu-server-post-installation.md   ← you are here
```

## Before You Begin
This guide assumes you have completed the **Ubuntu Server Installation** guide from the repo and are now ready to apply a clean, consistent post-install baseline.

---

# System Updates

Before doing anything else, update the system.  
Fresh installs often have several pending updates, and applying them now avoids troubleshooting issues that patches would have already solved.

Refresh package lists and install all pending updates:

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

This command does two things in sequence:

- `apt-get update` refreshes the package list so the system knows about the latest available versions.
- `apt-get upgrade -y` installs all available updates, and the `-y` flag pre-approves the changes so you're not prompted to confirm each one.

Chaining them with `&&` ensures the upgrade only runs if the update step completes successfully — a simple safety check that avoids half-completed operations.

---

# Enable Unattended Security Updates

Enable Ubuntu’s unattended security updates to keep the system patched automatically:

```bash
sudo apt-get install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

This enables **security-only** automatic updates, which is the safest and most predictable approach for a homelab server.

### Optional: Update Strategy Tuning

If you want finer control, Ubuntu allows adjusting the update behaviour:

- **Security-only (default):** applies critical security patches automatically  
- **Critical-only:** restrict updates to high-severity items  
- **Deferred updates:** delay non-critical updates for stability  
- **Manual full updates:** suitable if you snapshot before patching or maintain a change window  

For most homelab systems, the default **security-only unattended updates** is ideal.

---

# Install btop

A lightweight, real-time system monitor that helps you understand what the machine is doing during configuration or troubleshooting.

```bash
sudo apt-get install btop
```

---

# Configure Audit Logging

Audit logging provides lightweight visibility into key system events.

Install and enable the audit daemon:

```bash
sudo apt-get install auditd
sudo systemctl enable --now auditd
```

The default rule set is fine for a homelab server.

---

# Systemd Journal Retention Controls

Prevent `journald` from consuming excessive disk space.

Create directory:

```bash
sudo mkdir -p /etc/systemd/journald.conf.d
```

Create retention config:

```bash
sudo nano /etc/systemd/journald.conf.d/size.conf
```

Add:

```
[Journal]
SystemMaxUse=200M
SystemMaxFileSize=50M
```

Apply changes:

```bash
sudo systemctl restart systemd-journald
journalctl --disk-usage
```

After many troubleshooting sessions, I strongly recommend setting journal retention controls (log file limits) - it's more sustainable than continually expanding your virtual disk (and then planning which VM you're moving to another disk so you can accomodate the next). 

If you need to retain logs, offload them from `/` — We'll cover that in another guide.

---

# Kernel Hardening (sysctl)

Apply a small set of safe, high-value network and kernel protections.

Open or create the file:

```bash
sudo nano /etc/sysctl.d/99-hardening.conf
```

Add:

```
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.rp_filter = 1
kernel.kptr_restrict = 2
```

Apply:

```bash
sudo sysctl --system
```

---

# Summary Table for sysctl

| Setting | Purpose | Protection |
|--------|----------|------------|
| accept_source_route=0 | Block source-routed packets | Prevent path manipulation |
| accept_redirects=0 | Ignore ICMP redirects | Prevent rogue gateway attacks |
| send_redirects=0 | Stop sending redirects | Avoid routing interference |
| rp_filter=1 | Reverse-path validation | Blocks IP spoofing |
| kptr_restrict=2 | Hide kernel pointer addresses | Reduces exploitability |

---

# Reboot to Apply Everything

Some changes apply immediately, but others take effect only after a reboot.

```bash
sudo reboot
```

---

# Next Steps

Your server now has:

- current packages  
- automated security updates  
- predictable logging behaviour  
- capped journal retention  
- active audit logging  
- hardened networking defaults  

---

# Why These Choices Were Made

### Updating First
Ubuntu ISOs age quickly — even a “new” one can already be behind on security fixes or bug patches.  
Running updates immediately gives you:

- a clean, known-good baseline  
- predictable behaviour across deployments  
- fewer confusing issues caused by outdated packages  

Like clearing the workbench before starting a new build.

---

### Security-Only Unattended Updates
Full unattended upgrades can introduce unexpected behaviour, especially when major revisions land.  
Security-only updates strike the right balance:

- vulnerabilities get patched  
- the system stays stable  
- you maintain control of when larger updates happen (e.g., after a snapshot or maintenance window)

Essentially, it keeps us patched where it matters without adding extra complexity early on. 

Of course, if you prefer a different update cadence from the outset, implement it!

---

### btop
`top` works.  
`htop` works better.  
`btop` is the one you actually *want* to use.

It gives you:

- clearer layout  
- readable real-time graphs  
- CPU, RAM, disk and network IO at a glance  
- excellent visibility during early configuration  

Btop proves big things come in small (downloadable) packages!

---

### auditd
`auditd` quietly captures important security events:

- logins and authentication attempts  
- configuration edits  
- file access changes  
- privilege escalation  

This gives you historical visibility without the noise or overhead of a full SIEM solution — which is ideal for a homelab that wants accountability without complexity.

---

### journald Retention Caps
Left alone, Ubuntu will allow logs to expand until the entire OS disk is full.  
On small system disks, this can cause:

- containers stopping  
- services failing for “no reason”  
- failed boots because `/var` is full  

Capping log retention prevents problems before they happen.  
If long-term retention matters, offload logs — don’t expand the system disk.

---

### sysctl Hardening
These sysctl settings are safe defaults that:

- block spoofed packets  
- prevent malicious redirects  
- disable unused routing behaviour  
- hide kernel pointers from userland  
- reduce unnecessary network exposure  

They tighten security without changing how the server behaves day to day — minimum effort, maximum value.

---

### Rebooting Afterwards
Most changes apply immediately, but a few require a reboot to fully take effect.  
A restart ensures:

- updated kernel is live  
- journald limits are applied  
- auditd is initialised cleanly  
- all sysctl settings are committed  

And, of course, it’s an excellent excuse to get a coffee while the server “configures itself.”

# Recipe Building

This guide forms the **Post‑Installation Configuration** component for recipe building.
