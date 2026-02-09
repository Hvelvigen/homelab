# Proxmox VE 9.1 — Installation Guide
A clean, predictable, repeatable installation process for deploying Proxmox VE.

The goal is a minimal, boring hypervisor baseline that behaves the same every time — whether this is your first Proxmox install or your fiftieth.

Keyboard-only inputs appear in brackets; each `[]` represents a single key press.

---

## Repository Location

systems/
└── proxmox/
    └── proxmox-ve-installation.md        ← you are here
    └── proxmox-ve-post-installation.md

---

## Before You Begin

This guide assumes:

- **Hardware**
  - x86_64 CPU with hardware virtualisation support (VT-x / AMD-V)
  - IOMMU support available (VT-d / AMD-Vi recommended)
  - 8 GB RAM minimum (16 GB recommended)
  - One dedicated system disk (SSD or NVMe)

- **Firmware**
  - Virtualisation enabled
  - Boot mode set to UEFI
  - Secure Boot disabled (recommended for simplicity)

- **Installation media**
  - Proxmox VE 9.1 ISO written to USB

- **Networking**
  - At least one active network interface
  - DHCP available *or* known-good static network details

The installer will erase the selected system disk.
If that is not acceptable, stop here.

---

## Scope

This document covers **only** the base installation of Proxmox VE 9.1.

It does **not** include:
- Post-install updates
- Repository configuration
- Host hardening
- VM or container creation
- Storage or network tuning

Those are handled in separate guides.

---

## Installation Guide

### 1. Boot Installation Media

1. Insert the Proxmox VE USB installer.
2. Power on the system.
3. Use the system boot menu to select the USB device.
4. At the Proxmox boot menu, select:

   **Install Proxmox VE (Graphical)** → **[Enter]**

The graphical installer is recommended for most systems.

---

### 2. Licence Agreement

5. Review the licence agreement.
6. Select **I agree** → **[Enter]**

---

### 3. Target Disk Selection

7. Select the disk intended for the Proxmox OS.
8. Confirm the correct disk is selected.
9. Leave **Options** at defaults unless you have a specific requirement.

Defaults used:
- No ZFS
- No RAID
- No disk encryption

Select **Next** → **[Enter]**

**Note:**  
The selected disk will be wiped.

---

### 4. Location, Time Zone, Keyboard

10. Select:
    - Country
    - Time zone
    - Keyboard layout

11. Confirm selections → **Next** → **[Enter]**

These settings affect logs, certificates, and timestamps.
Correct values now prevent confusion later.

---

### 5. Administrator Credentials

12. Configure:
    - Root password
    - Administrative email address

The root account controls the entire host.
Use a strong password.

The email address is used for system notifications and alerts.

Select **Next** → **[Enter]**

---

### 6. Network Configuration

13. Select the primary network interface.
14. Configure:
    - Hostname
    - IP addressing
    - Gateway
    - DNS

Recommended approach:
- DHCP with a reservation at the router

Static addressing is also supported if your network requires it.

Confirm settings → **Next** → **[Enter]**

---

### 7. Review & Install

15. Review the summary screen carefully.

Confirm:
- Correct target disk
- Correct hostname
- Correct IP configuration
- Correct time zone

If anything is wrong, go back and correct it.

Select **Install** → **[Enter]**

---

### 8. Installation

16. Wait while Proxmox installs.

The installer will:
- Partition the disk
- Install the base OS
- Configure networking
- Install the web interface

No input required.

---

### 9. Reboot

17. When prompted:
    - Remove the installation media
    - Select **Reboot** → **[Enter]**

The system will reboot into the newly installed Proxmox host.

---

## First Boot Validation

After reboot, confirm at the console:

- Login prompt appears cleanly
- No boot warnings or errors
- The management IP address is displayed

The console will show a URL similar to:

  https://<host-ip>:8006/

From another machine on the same network:
- Open a browser
- Navigate to the URL
- Accept the self-signed certificate warning
- Log in as `root`

If the web interface loads, the installation is complete.

---

## Why These Choices Were Made

### Graphical Installer
The graphical installer:
- is well tested,
- reduces input errors,
- behaves consistently across hardware.

This guide optimises for repeatability, not novelty.

---

### Default Storage Layout
Using defaults:
- keeps the installation simple,
- avoids premature optimisation,
- aligns with common homelab usage.

More advanced storage layouts can be introduced later once requirements are clear.

---

### No ZFS by Default
ZFS is powerful, but it:
- increases memory requirements,
- adds complexity,
- is unnecessary for a basic hypervisor install.

It can be adopted later when the platform and workload justify it.

---

### DHCP or Known-Good Static Networking
Networking errors at install time are frustrating and silent.

Using DHCP (ideally with a reservation) ensures:
- predictable addressing,
- fewer initial failures,
- easier recovery.

---

## Next Steps

Continue with:

- **Proxmox VE – Post-Installation Configuration**

This is where updates, repositories, and baseline hardening are applied.

If the hypervisor is boring at this stage, it’s doing its job.
