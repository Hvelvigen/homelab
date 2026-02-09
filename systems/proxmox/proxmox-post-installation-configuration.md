# Proxmox VE 9.1 — Post-Installation Configuration
A clean baseline for a newly installed Proxmox VE host.

This guide applies a minimal, intentional post-install configuration focused on:
- stability
- security
- predictability
- long-term maintainability

The goal is to end up with a boring, trustworthy hypervisor that stays out of the way and does exactly what it’s told — no more, no less.

---

## Repository Location

    systems/
    └── proxmox/
        └── proxmox-ve-installation.md
        └── proxmox-ve-post-installation.md   ← you are here

---

## Before You Begin

This guide assumes:

- Proxmox VE 9.1 is installed and boots cleanly
- You can:
  - log in at the console as root
  - access the Web UI at https://<host-ip>:8006/
- No virtual machines or containers have been created yet

If the system is already in use, pause and decide whether you want to retrofit these changes or start fresh.
This guide assumes a clean host.

---

## Scope

This document covers:

- Essential host updates
- Repository configuration
- Safe, low-risk baseline settings
- Initial Web UI configuration

It does not cover:
- Advanced storage layouts
- Clustering
- Backup targets
- Firewall rule design
- VM or container creation

Those belong in later, more opinionated guides.

---

# Section 1 — Bash (Host-Level Configuration)

All steps in this section are performed from the Proxmox host shell, preferably:
- via SSH, or
- directly at the console if SSH access is not yet available

Proxmox enables SSH by default. Using it early keeps the workflow consistent with long-term host management - also, copy>paste ... IJS.

---

## 1. Configure Proxmox Repositories (No-Subscription)

By default, Proxmox enables the enterprise repository, which requires a subscription.

For homelab use, switch to the no-subscription repository before performing updates.

### Disable the Enterprise Repository

    nano /etc/apt/sources.list.d/pve-enterprise.sources

Then comment every line in the block by prefixing with #

### Disable the Ceph Repository

    nano /etc/apt/sources.list.d/ceph.sources

Then comment every line in the block by prefixing with #

---

### Enable the No-Subscription Repository

Create or edit the following file:

    nano /etc/apt/sources.list.d/pve-no-subscription.sources

Add:

    Types: deb
    URIs: http://download.proxmox.com/debian/pve
    Suites: trixie
    Components: pve-no-subscription
    Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg

Save and exit.

Validate with: 

    apt policy proxmox-widget-toolkit

---

## 2. Update the Host

With repositories correctly configured, update the system:

    apt update && apt full-upgrade -y

This ensures:
- security patches are applied
- kernel and core components are current
- updates are sourced from the correct repository

If a reboot is required, do it now:

    reboot

Reboots are cheap at this stage. They get expensive later.

---

## 3. Subscription Notice (Expected)

On Proxmox VE 9.1 without a subscription, the Web UI displays a
“no valid subscription” notice.

This is expected behaviour.

- It does not affect updates from the no-subscription repository
- It does not limit functionality
- It can safely be ignored

Proxmox no longer provides a stable mechanism to suppress this notice,
and attempting to do so requires modifying UI assets that are replaced
during updates.

For a homelab system, the notice is informational only.

---

## 4. Install Basic Host Tooling

Install a small set of useful utilities:

    apt install -y btop curl wget sudo

These help with:
- resource visibility
- diagnostics
- future automation

Nothing here runs persistently or alters system behaviour.

---

## 5. Time Synchronisation

Confirm time synchronisation is active:

    timedatectl status

You should see:
- System clock synchronized: yes
- NTP service: active

Correct time matters for logs, certificates, backups, and troubleshooting.

---

## 6. Final Reboot

If kernel or core packages were updated, reboot once more to ensure everything is live:

    reboot

After the system returns, host-level configuration is complete.

---

# Section 2 — Web UI Configuration

All steps in this section are performed via the Proxmox Web UI.

Log in at:

    https://<host-ip>:8006/

---

## 7. Verify Node Health

Select the node → Summary.

Confirm:
- CPU and memory are detected correctly
- No warnings or errors are present
- Uptime reflects the recent reboot

If something looks wrong here, stop and investigate before proceeding.

---

## 8. Storage Verification

Navigate to:

    Datacenter → Storage

Confirm:
- local exists (directory storage)
- local-lvm exists (LVM-thin)

Do not change storage layouts yet.
Defaults are intentional at this stage.

---

## 9. Network Verification

Navigate to:

    Node → System → Network

Confirm:
- vmbr0 exists
- It is bridged to the correct physical NIC
- The node IP matches expectations

Avoid additional bridges or VLANs for now.
Networking complexity compounds quickly.

---

## 10. DNS & Node Name Sanity Check

Navigate to:

    Node → System → DNS

Confirm:
- Hostname is correct
- DNS servers are reachable
- Search domain (if set) is intentional

These values affect updates, clustering later, and name resolution inside VMs.

---

## 11. UI Update Check

Navigate to:

    Node → Updates

Confirm:
- No pending critical updates
- The update channel reflects no-subscription usage

The UI should now be quiet and boring.
This is the desired state.

---

## 12. Create Administrative User Accounts

At this point, the host is stable and updated.  
Before creating workloads, create **normal administrative users** and stop using `root` for day-to-day access.

Root remains available as a break-glass account.  
It just shouldn’t be your daily driving seat.

---

### Create the Linux User (via Web UI Shell)

For users in the **Linux PAM** realm, the corresponding Linux user must exist on the host first.  
Proxmox does not create Linux users automatically.

To avoid context switching, this can be done directly from the Web UI.

Navigate to:

    Node → Shell

Create the user:

    adduser <username>

- Use the same username you intend to use in the Proxmox Web UI
- Set a strong password when prompted
- Accept defaults for the remaining questions

(Optional but practical) allow administrative commands without switching to root:

    usermod -aG sudo <username>

Exit the shell when complete.

At this point:
- the Linux user exists
- PAM authentication will succeed
- the account is ready to be linked to Proxmox permissions

---

### Create an Administrative Group

Proxmox permissions are easier to manage when assigned to groups rather than individual users.

Navigate to:

    Datacenter → Permissions → Groups

Click **Add** and create:

- **Group:** `Administrators`
- **Comment:** optional but recommended

Click **Add**.

---

### Assign Administrative Permissions to the Group

Navigate to:

    Datacenter → Permissions

Click **Add** → **Group Permission** and configure:

- **Path:** `/`
- **Group:** `Administrators`
- **Role:** `Administrator`

Confirm the assignment.

This grants full administrative access across the platform to any user in the group.

---

### Create the Proxmox User and Add to the Group

Navigate to:

    Datacenter → Permissions → Users

Click **Add** and configure:

- **Realm:** Linux PAM
- **User name:** the Linux username created earlier
- **Groups:** `Administrators`
- **Comment:** optional but recommended

Click **Add**.

No password is set here — authentication is handled by the underlying Linux account.

---

### Rationale

Proxmox treats `root` as a first-class control-plane account.  
Removing or disabling it causes friction later — especially during recovery, upgrades, or cluster operations.

The preferred model is:

- `root`
  - retained
  - rarely used
  - reserved for break-glass scenarios

- Administrative group
  - holds the `Administrator` role
  - provides consistent permissions
  - simplifies future user management

- Named admin users
  - members of the administrative group
  - used day-to-day
  - auditable and predictable

This mirrors good practice without fighting the platform.

---

### Optional: SSH Access

If you intend to use SSH with this account:

- Configure key-based authentication
- Prefer this account over `root` for remote access

SSH hardening and access policy belong in a later guide.

---

After this step, administrative access is no longer coupled to `root`.

Root remains available as a break-glass account, not a daily one.

---

## Exit Criteria

This phase is complete when:

- The host is fully updated
- No enterprise repository warnings remain
- Web UI loads cleanly
- Storage and networking are verified
- No VMs or containers exist yet

The hypervisor is now stable, predictable, and ready for workload deployment.

---

## Next Steps

From here, you can proceed to:
- VM creation
- Backup configuration
- Network refinement
- Project-specific workload design

Those decisions belong above the hypervisor.

At this level, the system should feel calm, uneventful, and slightly dull.

That’s success.
