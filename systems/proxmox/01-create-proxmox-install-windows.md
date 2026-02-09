# Create Proxmox VE 9.1 Installation Media (Windows 11 + Rufus)
A clean process for creating a bootable Proxmox VE USB installer from Windows 11.

---

## Repository Location

systems/
└── proxmox/
    └── proxmox-ve-installation-media.md   ← you are here
    └── proxmox-ve-installation.md

---

## Before You Begin

This guide assumes:

- You are running **Windows 11**
- You have **administrator rights** on the system
- You have:
  - A USB flash drive (8 GB minimum recommended)
  - The **Proxmox VE 9.1 ISO**
  - **Rufus** (current release)

**Important:**  
The USB drive will be erased completely.

If that is a problem, stop here and pick a different USB stick.

---

## What You’ll Need

### Proxmox VE ISO

- Download the **Proxmox VE 9.1 ISO installer** from the official Proxmox website:

  https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso/proxmox-ve-9-1-iso-installer

- **SHA-256 checksum (official):**
6d8f5afc78c0c66812d7272cde7c8b98be7eb54401ceb045400db05eb5ae6d22

You can verify this against the downloaded ISO to confirm integrity.

- Ensure the download completes successfully. If you verify the checksum, future-you will quietly approve.


---

### Rufus

- Download Rufus from: https://rufus.ie
- Use the portable release if prompted
- No installation is required; Rufus runs directly

---

## Create the USB Installer

### 1. Insert the USB Drive

1. Insert the USB flash drive into the Windows system.
2. Close File Explorer windows that may auto-open.

---

### 2. Launch Rufus

3. Right-click **Rufus** → **Run as administrator**
4. If prompted by UAC, select **Yes**

Running as administrator avoids permissions issues when writing boot records.

---

### 3. Select Target Device

5. Under **Device**, select the correct USB drive.

Double-check this step.
Rufus will happily erase the wrong disk if you tell it to.

---

### 4. Select the Proxmox ISO

6. Under **Boot selection**, choose:
   - **Disk or ISO image**
7. Click **Select**
8. Browse to and select the **Proxmox VE 9.1 ISO**

Rufus will automatically detect it as a Linux bootable image.

---

### 5. Image Mode Selection

9. When prompted:
   - Select **Write in ISO Image mode (Recommended)**
   - Click **OK**

Do **not** use DD mode unless you have a specific reason.
ISO mode works reliably for Proxmox on modern systems.

---

### 6. Partition Scheme & Target System

Rufus should auto-populate the correct values.

Confirm the following:

- **Partition scheme:** GPT
- **Target system:** UEFI (non CSM)
- **File system:** FAT32
- **Cluster size:** Default

If Rufus selects these automatically (it usually does), leave them unchanged.

This aligns with modern UEFI-only systems and avoids legacy boot confusion.

---

### 7. Volume Label (Optional)

10. Optionally set:
    - **Volume label:** `PROXMOX_VE_9_1`

This is cosmetic, but helpful when multiple USB installers exist.

---

### 8. Start Writing

11. Click **Start**
12. If warned about data loss, confirm

Rufus will:
- Format the USB
- Copy the ISO contents
- Make the device UEFI-bootable

---

### 9. Wait for Completion

13. Wait until Rufus shows **READY**
14. Click **Close**
15. Safely eject the USB drive

Do not remove the USB early.
Partial writes result in very confusing boot behaviour later.

---

## Validation

Before moving on:

- The USB drive shows files when viewed in File Explorer
- Rufus completed without errors
- The USB drive is safely ejected

At this point, the installer media is ready.

---

## Common Pitfalls (Avoid These)

- Using **MBR** instead of GPT  
  → Causes boot failures on UEFI-only systems

- Using **DD mode** unnecessarily  
  → Makes the USB harder to inspect and debug

- Leaving **CSM / Legacy boot** enabled in firmware  
  → Can result in the system booting the wrong path

- Writing the ISO from a corrupted download  
  → Results in silent install or boot failures

---

## Next Steps

Proceed to:

- **Proxmox VE Installation Guide**

Insert the USB drive into the target system, select it from the boot menu, and begin installation.

If the system boots straight into the Proxmox installer menu, the media creation step was successful.

---

## Why This Approach

Rufus + ISO mode:
- is reliable
- is widely tested
- works cleanly with Proxmox VE ISOs
- avoids legacy compatibility traps

The goal is not flexibility — it’s predictability.

If the USB installer boots cleanly every time, it has done its job.
