# Ubuntu Server 24.04.3 LTS — Installation Guide
A clean, predictable, repeatable installation process for deploying Ubuntu Server in a homelab.
The goal is a minimal baseline that behaves the same every time — whether this is your first install or your fiftieth.

Keyboard-only inputs appear in brackets; each `[]` represents a single key press.

---

## Repository location

```
systems/
└── linux/
    └── ubuntu/
        └── ubuntu-server-installation.md   ← you are here
```
---

## Before you begin
This guide assumes:

- **System requirements**:
  - 1 vCPU (2+ recommended)
  - 1–2 GB RAM minimum (4 GB recommended)
  - System disk: 16–32 GB
  - Data disk: sized for your service workloads
  - Reliable LAN/VLAN connectivity

- The VM has **two disks**:
  - **System disk** — OS only
  - **Data disk** — Docker, apps, logs, anything persistent  
    *Only the system disk is touched during installation.*

- VLANs, routing, and DNS resolution have already been validated at the hypervisor layer.

- No encrypted root or LVM is required for this baseline.

---

## Scope
This document covers the base installation of Ubuntu Server 24.04.3 LTS.  
Post-install configuration, hardening, and service setup are handled in separate guides.

---

## Installation guide

1. Start the VM.

2. Let GRUB auto-select, or press **[Enter]** on *“Try or Install Ubuntu Server.”*

3. Wait for the installer to load.

4. Select your **language/locale** → **[Enter]**.

5. Confirm keyboard layout → **[Enter]**.

6. Ensure **Ubuntu Server** is selected → **Done** → **[Enter]**.

---

### Network configuration

7. Network Configuration depends on your environment, you may need to set this manually if using non-DHCP reserved static assignments (hello Excel!).
   Choose the method appropriate for your environment:

  #### 7.1 DHCP (no reservation) — *not recommended*
  - Simply select **Done** → **[Enter]**.
  
  #### 7.2 Manual IP assignment
  - Highlight the network adapter → **[Enter]**
  - Select **Edit IPv4** → **[Down][Down][Enter]**
  - Change IPv4 Method: **Manual** → **[Down][Enter]**
  - Enter addressing details (`Subnet` uses CIDR)
  - **[Tab]** → **Done** → **[Enter]**
  
  #### 7.3 DHCP reservation — *recommended*
  - Confirm the address matches your IP schema.

8. **Done** → **[Enter]**.

9. Leave the proxy empty → **[Tab][Enter]**.

---

### Mirror test

10. If networking is correct, the mirror test passes.  
Select **Done** → **[Enter]**.  
If not, revisit Step 7 and try again — Ubuntu is usually right when it says something can’t reach the internet.

---

### Storage configuration

11. In the storage screen:
    - Use **Use an entire disk**
    - Select the system disk
    - **Uncheck**: *Set up this disk as an LVM group* → **[Tab][Tab][Space]**
    - Select **Done** → **[Enter]**

12. Review the layout:
    - `/`
    - `/boot/efi`  
    → **Done** → **[Enter]**

13. Confirm changes → **Continue** → **[Down][Enter]**

---

### Profile setup

14. Enter your name, server name, username, password.  
Use **[Tab]** → **Done** → **[Enter]**.

---

### Ubuntu Pro

15. Skip Ubuntu Pro for now — it adds no value at this early stage.

---

### SSH and snaps

16. Skip SSH (we configure it properly in post-install) → **[Tab][Enter]**

17. Skip snaps → **[Tab][Enter]**

---

### Finalisation

18. Wait...

19. When prompted:
    - Select **Reboot Now** → **[Tab][Down][Enter]**
    - Detach the ISO/USB from the VM  
      Ubuntu will briefly complain about failing to unmount the device — that’s normal, and it’s your cue to detach the installation media.

20. The system will reboot to the **login:** prompt and is ready for post-installation tasks.

---

## Why these choices were made

### Default “Ubuntu Server”
The installer’s default **Ubuntu Server** selection is intentionally used here because it provides:
- a clean, predictable server base,
- a well-supported package set,
- and none of the desktop or unnecessary service overheads.

### Two-Disk layout
Using a dedicated **system disk** keeps the OS self-contained and disposable.  
The **secondary disk** becomes the home for Docker, service data, logs, and anything designed to persist across rebuilds.

The advantages:
- Rebuilding the OS does not risk wiping your containers or data  
- Maintenance stays tidier  
- Storage layout stays consistent regardless of VM role  
- You avoid Ubuntu scattering persistent data across `/var` on the system disk

### No LVM
LVM is useful for environments where you extend volumes dynamically or juggle physical disks.  
In a homelab VM:
- disks are virtual,
- expansion is done at the hypervisor layer,
- and complexity slows everything down for no practical benefit.

Disabling LVM gives you:
- a simpler partition scheme,
- less abstraction,
- and an easier time snapshotting, backing up, or rebuilding.

### Networking approach
Networking is one of the easiest ways an installation can go sideways.

The guide supports three approaches because different homelabs behave differently:
- **DHCP (temporary)** when you just need connectivity  
- **Manual** when you want predictable addressing  
- **DHCP reservation** when you want predictability *and* set-and-forget convenience  

Reserved/static addressing keeps the system exactly where you expect it to be, avoids future reconfiguration, and aligns with IP schemas you control (well.. hello again, Excel).

Use the method that fits your goals.

### Skip SSH
The installer enables SSH with defaults you don’t control — and those defaults often don’t match a good baseline hardening.

Skipping SSH now means:
- you install it cleanly after first boot,
- you apply your own security baseline,
- and you avoid Ubuntu silently enabling features you weren’t ready for.

It’s not skipped because SSH is “bad”; it’s skipped so *you* decide how it starts.

### Skip snaps
Snaps:
- auto-update,
- operate under confinement rules,
- and place runtime data in non-standard paths.

This creates unpredictable behaviour when trying to maintain a reproducible homelab environment.

Skipping snaps:
- keeps things cleaner,
- ensures file paths are where you expect them,
- and prevents packages updating themselves out of sync with the rest of the system.

If you want snaps later, you install them intentionally — not because the installer offered them.

### Skip Ubuntu Pro
Ubuntu Pro is useful, but not required for establishing a clean baseline.

By skipping it during installation:
- you avoid mixing subscription and token flow with setup,
- you ensure the base OS works consistently without external dependencies,
- and you decide later if you actually want Pro features.

It keeps the *first build* simple and reproducible.

### Detail level
The goal is clarity for all experience levels.  
Someone installing Ubuntu for the first time should be able to follow this without stress.  
Someone experienced should be able to skim the headings and still know exactly what the installer will do.

It also helps **future me** — who will inevitably revisit this and ask,
“Why did I set it up like this?”

This section is here so I don’t have to make eye contact with myself while answering.

---

## Next steps
The OS is installed.  
Continue with post-install configuration, hardening, and service deployment.

---

## Recipe Building
This guide forms the **Base Operating System** component for recipe-style builds.
