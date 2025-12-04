# Linux Reference — Disk Management Essentials
*A practical guide for working with disks, partitions, filesystems, and mounts in Ubuntu Server.*

This reference pulls together the core commands and patterns you’ll actually use when working with storage in your homelab — from spotting new disks, prepping and formatting them, to mounting them cleanly and keeping an eye on their health.

It’s written for anyone comfortable in a terminal but not necessarily living and breathing Linux storage.

---

## 1. Inspecting Disks & Block Devices
Before touching a disk, you need to understand what the system can actually see.  
These commands help you discover disks, check their layout, and confirm what’s already mounted so you don’t accidentally modify the wrong device.

And remember, before running anything destructive, make sure you’re looking at the right disk — lsblk is your friend here.

### List block devices
```
lsblk
```
Shows block devices, their partitions, sizes, and mount points.  
Useful for spotting new disks or confirming which partition is mounted where.

Useful variations:
```
lsblk -f
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
```
Adds filesystem type, UUIDs, and clearer structure — essential when preparing fstab entries.

### Detailed device info
```
sudo fdisk -l
```
Provides deeper information: partition tables (GPT/MBR), exact sectors, and boot flags.  
Helpful when diagnosing disks that won’t format or mount correctly.

### Show mounted filesystems
```
df -h
```
Displays mounted filesystems and how much space they’re using.  
Use this to verify temporary or permanent mounts and check storage health.

### Identify a newly added disk
Look for a device with **no partitions and no filesystem**:
```
/dev/sdb
/dev/sdc
```
New disks will appear here with no FSTYPE in `lsblk`.

---

## 2. Partitioning a New Disk (GPT)
A disk must be partitioned before creating a filesystem. GPT is standard for modern systems, reliable, and supported everywhere.

GPT behaves consistently across modern systems, which is lovely when you just want things to work without the occasional MBR chaos of decades past (/wave BCDEdit).

### Using parted
```
sudo parted /dev/sdb
mklabel gpt
mkpart primary ext4 0% 100%
quit
```
Creates a GPT partition table and one full‑disk partition.  
`ext4` here is only a label — the actual filesystem comes later with `mkfs`.

**ZFS** considerations are covered in dedicated docs.

### Using fdisk (interactive)
```
sudo fdisk /dev/sdb
g
n
  - The default value '1' is just fine.
  - Use the default value for First and Last sector.  
w
```
`g` → create GPT table  
`n` → new partition  
`w` → write changes  
Ideal if you prefer a step-by-step interactive workflow.

---

## 3. Creating a Filesystem
A partition isn’t usable until it has a filesystem.  
Ext4 is the safe default for Ubuntu Server — mature, robust, and fast.

```
sudo mkfs.ext4 /dev/sdb1
```
Creates an ext4 filesystem on the new partition.

```
sudo e2label /dev/sdb1 docker-data
```
Adds a human-readable label, helpful when managing many disks or troubleshooting mounts.

---

## 4. Mounting a Disk (Temporary)
Temporary mounts let you test a disk before committing it to `/etc/fstab`.

They're also a nice way to sanity-check a drive before committing it to your long-term layout — a quick “does this behave?” test.

```
sudo mkdir -p /mnt/testdisk
```
Creates a mount point.

```
sudo mount /dev/sdb1 /mnt/testdisk
```
Attaches the filesystem.

```
df -h | grep sdb1
```
Confirms the mount.

```
sudo umount /mnt/testdisk
```
Cleanly detaches it — always unmount before editing fstab or resizing partitions.

---

## 5. Permanent Mounting with /etc/fstab
Temporary mounts vanish on reboot.  
`fstab` is powerful — but unforgiving. It lets you make mounts persistent — but mistakes here can render your system unbootable. 
A single typo can ruin your morning, so always test with mount -a before rebooting. Your uptime (and sanity) will thank you.

### Get UUID
```
lsblk -f
```
UUIDs are stable identifiers; device names like `/dev/sdb` can change between boots.

### Create mount point
```
sudo mkdir -p /srv/docker
```
A clean, standard location for persistent service data.

### Edit fstab
```
sudo nano /etc/fstab
UUID=<uuid>  /srv/docker  ext4  defaults  0  2
```
This tells Ubuntu to mount the filesystem at boot.

### Test before reboot
```
sudo mount -a
```
If nothing prints: success.  
If errors appear: fix before rebooting.

---

## 6. Disk Health & SMART
SMART gives insight into physical drive health — invaluable for catching problems early, there is no value on a virtual hard disk.

Install tools:
```
sudo apt-get install smartmontools
```

Check full SMART output:
```
sudo smartctl -a /dev/sdb
```

Run a short test:
```
sudo smartctl -t short /dev/sdb
```

---

## 7. Disk Usage & Growth
Track disk usage so you don’t run out of space unexpectedly.

Directory total:
```
du -sh /srv/*
```

Find largest directories:
```
du -h --max-depth=1 /srv | sort -h
```

---

## 8. Expanding Disks (VM-based)
Virtual disks grow easily — the OS just needs to be told.
Before attempting the below, you'll need to expand the virtual disk in your hypervisor.

Rescan disk:
```
sudo partprobe
echo 1 | sudo tee /sys/class/block/sdb/device/rescan
```

Grow partition:
```
sudo growpart /dev/sdb 1
```

Resize filesystem:
```
sudo resize2fs /dev/sdb1
```

---

## 9. Mount Troubleshooting
Mount issues are usually misconfigurations or missing paths.

Check system logs:
```
sudo journalctl -xe | grep mount
```

Validate fstab:
```
sudo mount -a
```

Verify disk visibility:
```
lsblk
```

---

## 10. Example Layouts Common in Homelabs
Standardising disk layouts keeps deployments repeatable.

### Dedicated disk for containers
```
/srv/docker/
```

This layout assumes the host is using the Docker Host Service Layer (/srv/host-docker). See HSL docs for full structure.
Use this as the parent directory for all container data.

### Multi-service layout
```
mkdir -p /srv/docker/{it-tools,unifi,grafana}
```
Each service gets its own folder — cleaner backups and troubleshooting.

### Remote storage
Use bind mounts for NAS-backed data, not for core Docker volumes.  
Helps isolate performance and permission differences.

---

## 11. Danger Zone
A few high-risk actions to treat with caution:

- `mkfs` on the wrong device = instant data loss  
- Incorrect fstab entry = system fails to boot  
- NTFS/exFAT = permission issues and broken containers  

If you’re not sure — stop, check `lsblk`, and confirm paths first.

---

## 12. Quick Reference Table

| Task | Command |
|------|---------|
| List disks | `lsblk` |
| Partition | `parted /dev/sdb` |
| Format ext4 | `mkfs.ext4 /dev/sdb1` |
| Mount | `mount /dev/sdb1 /mnt` |
| Unmount | `umount /mnt` |
| UUID | `lsblk -f` |
| Test fstab | `mount -a` |
| SMART | `smartctl -a /dev/sdb` |
| Resize FS | `resize2fs /dev/sdb1` |

---

## 13. Summary
This guide provides a clear, repeatable baseline for working with disks on Ubuntu Server.  
It supports the storage workflows you’ll use when building Docker hosts, deploying services, and maintaining a tidy, predictable homelab environment.
This keeps everything tidy and predictable. The separation of concerns is a nice bonus.
