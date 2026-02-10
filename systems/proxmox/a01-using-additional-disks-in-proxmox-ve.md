# A01 – Using Additional Disks in Proxmox VE (PVE 9.1)

This guide describes the **Proxmox-native way** to use additional physical disks attached to a Proxmox VE 9.1 host.

It is intentionally:
- single-node focused  
- boring  
- reversible  

The goal is to make extra disks **useful without turning the hypervisor into a NAS or binding storage to a single VM by accident**.

---

## Scope

This guide applies to:

- Proxmox VE **9.1**
- Single-node deployments
- One or more **additional physical disks** (HDD or SSD)

This guide **does not** cover:

- ZFS pools
- RAID (hardware or software)
- Ceph
- Clustering
- SMB / NFS configuration
- Backup policy or retention
- VM- or OS-specific disk layout decisions

Those belong in separate, more opinionated guides.

---

## The Proxmox storage mental model

Before touching any disks, it’s important to align on how Proxmox expects storage to work.

**Proxmox owns physical disks.**  
**Virtual machines consume storage that Proxmox deliberately exposes.**

This means:

- Physical disks are managed **at the host level**
- VMs receive **virtual disks**, not raw hardware by default
- Storage survives VM rebuilds
- The hypervisor remains boring and predictable

Mounting physical disks directly inside a VM bypasses this model and creates avoidable coupling.  
This guide deliberately avoids that pattern.

---

## When you should add an additional disk

Common, valid reasons include:

- a backup target for Proxmox VMs
- bulk or cold storage
- additional capacity for VM data disks
- temporary or transitional storage
- separating “large and slow” data from primary VM storage

Common **invalid** reasons:

- “so Ubuntu can mount it directly”
- “to act like a NAS”
- “because Docker needs it”
- “because I might need it later”

Additional disks should be added with **intent**, not anticipation.

---

## Disk discovery and verification

Before initialising anything, confirm the disk exists and is not already in use.

### Via the Web UI

Navigate to:

    Node → Disks

Confirm:
- the disk appears with the expected size
- it is not part of an existing storage configuration
- it is not holding data you care about

### Via the shell (optional sanity check)

    lsblk

You should be able to clearly identify:
- the system disk
- the additional disk(s)

If there is any uncertainty at this stage, stop.

---

## Choosing a Proxmox storage type

Proxmox supports multiple storage backends. For most homelab and single-node use cases, **directory storage** is the correct default.

### Directory storage (recommended)

- Flexible
- Human-readable
- Easy to recover
- Supports:
  - VM backups
  - ISO storage
  - VM disk images (qcow2 / raw)
- Simple to reason about

### LVM-thin (acceptable, but more opinionated)

- Efficient space usage
- Faster snapshots
- Less transparent
- Harder to recover outside Proxmox

### ZFS

Out of scope for this guide.

If you are not explicitly choosing ZFS for known reasons, you should not be using it here.

---

## Initialising the disk

All disk preparation is performed **on the Proxmox host**, not inside a VM.

### Step 1 — Wipe existing data

In the Web UI:

    Node → Disks

- Select the disk
- Choose **Wipe Disk**

This ensures Proxmox starts from a known state.

---

### Step 2 — Create the filesystem and mount point

Still under:

    Node → Disks

- Select the disk
- Choose **Initialize Disk with GPT** (unless you have a compelling reason not to)

Once the disk is initialised, switch to:

    Node → Disks → Directory → Create: Directory

Configure:

- **Disk:** select the newly initialised disk
- **Filesystem:** `ext4` (recommended)
- **Name:** a clear, descriptive identifier (e.g. `bulk-data`)

This step:
- creates the filesystem,
- mounts it persistently,
- and registers it cleanly with the host.

Avoid clever partitioning here.  
One disk → one filesystem → one purpose.

---

## Adding the disk as Proxmox storage

Once the disk is mounted:

Navigate to:

    Datacenter → Storage → <new Directory) → Edit

Configure:

- **Content:** select only what you intend to use
  - VM backups
  - Disk images
  - ISO images

Apply the configuration.

At this point, Proxmox fully owns the disk.

---

## Intended usage patterns

Additional disks managed this way are suitable for:

- Proxmox VM backups
- large VM data disks
- cold or bulk storage
- temporary capacity
- transitional setups

The key characteristic is **flexibility**.

---

## Explicit non-usage patterns

This storage is **not** intended to be:

- mounted directly inside the Proxmox host for services
- shared directly to Windows or other clients
- treated as irreplaceable or authoritative
- tightly coupled to a single VM’s lifecycle

If you need OS-level layouts, permissions, or application-specific structure, that belongs **inside the VM**, on a virtual disk backed by this storage.

---

## Relationship to virtual machines

The correct pattern is:

1. Proxmox manages the physical disk
2. Proxmox exposes storage capacity
3. VMs receive virtual disks
4. OS-level decisions happen inside the VM

Examples:
- Ubuntu Server receives a virtual disk backed by `bulk-data`
- Docker layouts are configured *inside the VM*
- Windows access is mediated by services, not raw disks

This keeps boundaries clean and future changes cheap.

---

## Validation checklist

After completing this process, the following should be true:

- [ ] The disk appears under **Datacenter → Storage**
- [ ] The mount survives a reboot
- [ ] Proxmox can write to the storage
- [ ] No VMs directly reference physical devices
- [ ] You can remove or rebuild a VM without losing the storage configuration
- [ ] The hypervisor remains boring

If all of these are true, the disk is configured correctly.

---

## Summary

Using additional disks in Proxmox VE is not about maximising performance or clever layouts.  
It is about **clear ownership, predictable behaviour, and clean separation of responsibility**.

When Proxmox owns the disk:
- VMs become disposable
- storage becomes reusable
- rebuilds become cheap
- mistakes become reversible

That is the entire point of a hypervisor.

If the host feels uneventful after this process, it’s doing exactly what it should.
