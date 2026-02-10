# OPNsense 26.1 — Installation Guide

A clean, predictable installation process for deploying **OPNsense 26.1** as a virtual machine.

The goal is to establish a **stable firewall baseline** that boots reliably, installs cleanly, and is easy to reason about — without premature optimisation or platform-specific hacks.

This guide covers **installation only**.  
Initial configuration, interface assignment strategy, and firewall policy are handled separately.

---

## Scope

This document covers:

- Booting the OPNsense 26.1 installer
- Performing a clean install to disk
- Verifying first boot

It does **not** cover:

- Virtual machine creation or hypervisor configuration
- Interface role design (WAN / LAN / Wi-Fi)
- Firewall rules
- NAT, DHCP, DNS
- VPNs or remote access
- Performance tuning

Those belong above the installation layer.

---

## Assumptions

This guide assumes:

- A supported hypervisor or virtualisation platform is in use
- Hardware virtualisation support is available to the guest
- At least **two network interfaces** are presented to the VM
- You are installing **OPNsense 26.1 (latest stable)**
- A suitable virtual machine has already been created for OPNsense

---

## Installation Process (Inside OPNsense)

### 1. Boot the VM

Start the virtual machine and open the console.

The OPNsense boot menu should appear automatically.

If prompted:
- Press **Enter** to boot normally

---

### 2. Login to Installer

At the console prompt:

- **Username**: `installer`
- **Password**: `opnsense`

---

### 3. Start Installer

Select:

- **Install (UFS)**

ZFS is not required for a VM firewall and adds unnecessary complexity at this stage.

---

### 4. Keyboard Layout

- Select your preferred layout  
  (Default `US` is acceptable if unsure)

---

### 5. Disk Selection

- Select the **single virtual disk**
- Confirm the installation target

All data on the disk will be erased.

---

### 6. Swap

- Accept the default swap configuration

---

### 7. Install

Confirm the installation.

Wait for the disk copy process to complete.  
This typically takes a few minutes.

---

### 8. Root Password

Set a **strong root password**.

This controls:
- Console access
- Initial Web UI access

---

### 9. Complete Installation

When prompted:

- Select **Complete Install**
- Power off the virtual machine

---

## Post-Install Boot

### 1. Detach Installation Media

Remove or detach the OPNsense installation ISO from the virtual machine.

This step is required to ensure the system boots from disk rather than re-entering the installer.

---

### 2. Boot from Disk

Start the virtual machine again.

OPNsense should boot directly into the installed system.

---

## First Boot Validation

On successful boot, you should see:

- Interface detection messages
- The OPNsense console menu
- No boot errors or warnings

At this stage:
- Interfaces are **not yet assigned**
- Networking is **not yet active**

This is expected behaviour.

---

## Exit Criteria

This installation phase is complete when:

- OPNsense boots cleanly from disk
- The console menu is accessible
- Installation media is no longer attached
- No firmware or boot warnings are present

The system is now ready for:

- Interface assignment
- Initial configuration
- Firewall policy design

---

## Notes

- Secure Boot is intentionally not assumed
- Defaults are preferred unless a deviation is required
- Performance tuning belongs **after** functional validation

A firewall that boots reliably is more valuable than one that is theoretically optimal.

---
