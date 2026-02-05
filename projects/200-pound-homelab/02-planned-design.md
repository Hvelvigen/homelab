# 02 – Planned Design

This section captures the *intended* design of the Budget Home Server **before** any hardware is powered on or software installed.

This stage makes the most sense once you’ve narrowed things down to a small shortlist — or, realistically, once you’ve actually bought something and need to sanity-check how it *should* be used.  
It’s deliberately pragmatic: second-hand hardware doesn’t always arrive exactly as advertised, and designs should survive a bit of surprise RAM, a smaller SSD, or a drive you weren’t expecting.

The aim here is to be **holistic but flexible** — enough structure to guide decisions, without pretending this is a fixed blueprint.

---

## Design goals

Before thinking about VMs, containers, or diagrams:

- **What should this server be good at on day one?**  
  Being a simple-to-use, secure, and reasonably hardened **Proxmox VE host**, running a mix of containers/LXC and VMs. One of those VMs is almost certainly going to be **Home Assistant**, because ... of course it is.

- **What should it explicitly not try to do yet?**  
  Act as a NAS or a mass-storage solution, or take on heavy workloads such as serious AI inference. That’s a different problem space and a different budget.

- **What would “success” look like after the first week of use?**  
  The system is stable, services are running without incident, and the platform has already been expanded on the *software* side without anything feeling fragile or rushed.

---

## Host platform

- **Why is Proxmox VE the right choice for this build?**  
  It’s free and open source, well-supported, and something I’ve been using for around four years. For homelab use in particular, features like **USB passthrough** (for things such as Zigbee dongles) make it practical without getting in the way.

- **What alternatives were considered, and why were they rejected?**  
  **TrueNAS SCALE** was considered, but this isn’t a mass-storage system. It also tends to oversimplify certain concepts, which is counterproductive when part of the goal is learning how things actually fit together.

- **What assumptions are being made about uptime, reboots, and maintenance?**  
  The expectation is that this box only falls over during power outages or deliberate reboots. Routine maintenance should be as automated as possible — including a scripted approach to OS updates inside Ubuntu guests, using a scheduled and deferred model so updates happen *on purpose*, not as a surprise.

---

## Virtualisation approach

This is the first point where *I think* there isn’t a single “correct” answer — only trade-offs.

At a high level, the question isn’t just *VMs vs containers*, but **where containers should live**:
- Directly on the Proxmox host (LXC), or
- Inside a dedicated Linux VM running Docker

Both are valid. Both are common. And both show up repeatedly in homelab writeups for good reason.

For this project, the priority is **simplicity and flexibility**, not absolute efficiency or theoretical purity.

That means:
- Fewer sharp edges early on
- Clear separation between the host and workloads
- The ability to change direction later without rebuilding everything

Some services could reasonably run either as VMs or containers, and that decision doesn’t need to be locked in yet. What matters more at this stage is avoiding a design that’s clever on paper but brittle in practice.

Given those priorities, the primary choice for this build is **Docker inside a dedicated Ubuntu Server VM** for most application workloads, with LXC on Proxmox available later for small, self-contained utilities if it makes sense.

---

## Initial VM plan

Rather than aiming for maximum density, the initial plan is to keep things intentionally modest.

- **Expected VMs at the start:**
  - **Home Assistant** (of course)
  - **Ubuntu Server** (general-purpose services, Docker, experimentation)
  - **OPNsense** (stretch goal, optional — requires either an additional NIC or a USB dongle)

- **Roles (loosely defined):**
  - Home Assistant: home automation platform
  - Ubuntu Server: a flexible services host
  - OPNsense: optional firewall for testing or remote deployment

- **What’s optional or “later”:**
  - OPNsense is not required for day one
  - Home Assistant could move between hardware or platforms depending on future needs — it’s that flexible

At this stage, rough intent is enough.

---

## Container strategy

For this build, containers are intended to handle the bulk of the “always-on” application services — things that benefit from being lightweight, easy to update, and simple to tear down or move if needed.

### Services intended to run in containers (day one)
Based on the scope defined earlier, the following services make sense to containerise from the outset:

- Nginx Proxy Manager  (NPM)
- Pi-hole  
- Nextcloud  
- Frigate  
- Minecraft server  
- Jellyfin  
- Uptime Kuma  
- Speedtest Tracker  

These are all well-supported in container form, have mature images available, and don’t require tight coupling to the underlying hypervisor.

### Where containers will live
For this project, containers will run **inside a dedicated Ubuntu Server VM using Docker**.

This keeps a clean separation between:
- The **hypervisor** (Proxmox, kept intentionally boring), and
- The **workloads** (Docker containers, free to change and evolve - *really* useful when something goes oh so wrong!)

### Why this fits the project goals

- Docker-in-a-VM is a very common pattern with a large support ecosystem
- The services VM can be snapshotted, backed up, or migrated as a single unit
- Container definitions are portable if the platform changes in the future
- It avoids blurring the line between “host” and “workload” early on

In short, this trades a small amount of efficiency for clarity, portability, and fewer sharp edges — a sensible exchange for a starter homelab that’s meant to grow with its owner.

---

## Storage layout

The OptiPlex 5070 in this build comes with a **256 GB NVMe SSD** and a **500 GB HDD**. That’s not a lot by modern “media hoarder” standards, but it’s more than enough for a starter homelab if it’s used deliberately.

### Disk roles

- **Proxmox OS**
  - Lives on the **256 GB NVMe SSD**
  - Proxmox itself doesn’t need much space, but fast storage here keeps the host responsive

- **VM disks**
  - Primary VM disks (Home Assistant, Ubuntu Server, OPNsense if used) live on the **NVMe**
  - We want to keep anything latency-sensitive or “feels slow if the disk is sad” on flash

- **Container data**
  - Docker (inside the Ubuntu Server VM) also uses virtual disks backed by the **NVMe**
  - Databases, config, and app data for the containerised services sit here as well

- **HDD usage**
  - The **500 GB HDD** is reserved for:
    - Less performance-sensitive data
    - Lab files, test payloads, temporary dumps
    - Potential staging area for future backup experiments
  - Not treated as a NAS and not expected to be heroic

### Data importance

- **Important (should not vanish casually):**
  - Proxmox configuration
  - VM disks (especially Home Assistant and OPNsense, if in use)
  - Docker volumes and configs for core services (Nextcloud, Jellyfin, NPM, etc.)

- **Tolerable to lose or recreate:**
  - Test VMs
  - Throwaway containers
  - Logs that aren’t explicitly earmarked for retention
  - Temporary data parked on the HDD during experiments

Nothing here is truly “mission critical”, but losing certain pieces would still be irritating enough to justify some care.

### What isn’t being solved (yet)

On purpose, this design does **not** attempt to handle:

- Proper off-site or versioned **backups**
- **Redundancy** (no mirroring, no ZFS pool, no RAID)
- Large-scale media storage or multi-terabyte datasets

Those belong to a future NAS or a more storage-focused build.  
For now, we want a clear, fast layout that doesn’t pretend a single SFF box is secretly an entire storage strategy - ambition is good, physics is stubborn.

---

## Compute allocation (initial thinking)

At this stage, the goal isn’t to squeeze every last drop of performance out of the hardware — it’s to leave enough slack that the system stays pleasant to use and forgiving when ideas inevitably change.

### Headroom first, optimisation later

On day one, a deliberate chunk of CPU and RAM will be left **unallocated**:
- Enough CPU to ensure the host stays responsive
- Enough RAM to spin up a test VM or container without immediately having to reshuffle everything

Exact numbers aren’t critical yet, but if the host looks “full” on day one, something has gone wrong.

### Workloads that need guaranteed performance

Some services benefit from predictability more than raw throughput:
- **Home Assistant** should feel responsive at all times
- The **Ubuntu Server** VM (hosting Docker containers) should have stable resources to avoid noisy-neighbour behaviour between services

These will get sensible, fixed allocations rather than being left to fight it out.

### Rough resource expectations (not promises)

The table below reflects **reasonable starting points**, not hard limits.  
Think of this as “what keeps things comfortable”, not “what it will consume forever”.

| Workload | Minimum | Recommended | Notes |
|---|---|---|---|
| Proxmox host | 1 core / 2 GB RAM | 2 cores / 4 GB RAM | Keeps the host calm and responsive |
| Home Assistant VM | 2 cores / 2 GB RAM | 2 cores / 4 GB RAM | Benefits from low latency more than raw power |
| Ubuntu Server (Docker) | 2 cores / 4 GB RAM | 4 cores / 8 GB RAM | Shared by multiple services |
| OPNsense (optional) | 1 core / 1 GB RAM | 2 cores / 2 GB RAM | Only needed if used as a firewall |

These numbers are intentionally conservative and leave room for:
- Experimentation
- Short-lived test workloads
- The inevitable “just trying something” VM

### Where contention is acceptable

Not everything needs to be precious.

The following are acceptable places for contention *if* it happens:
- Test containers
- Non-critical services
- Anything experimental or clearly marked as “lab”

If something slows down here, that’s a signal — not a failure. It either needs more resources, better placement, or to be turned off entirely.

In short, the system is designed to feel **calm under normal use**, with enough flexibility left over to experiment without immediately repainting the whole house.

---

## Networking assumptions

Networking is intentionally kept simple for this build. The aim is to get useful services running reliably, not to turn the first iteration into a networking lab by accident.

### NICs and connectivity

This is **effectively a single-NIC design to start with**.

The OptiPlex includes:
- One physical Ethernet NIC

At this stage, there’s no firm commitment to purchasing an additional NIC. The hardware supports it, and the option remains open, but it isn’t a prerequisite for day one.

If and when a second NIC is added, that decision will be driven by an actual need rather than future-proofing for its own sake.

### VLANs

VLANs are **explicitly out of scope** for the initial build.

This project isn’t trying to model a full segmented network from day one. VLANs may appear later as part of learning or expansion, but they are not required to meet the core goals of this server.

### What complexity is intentionally avoided

For now, the following are deliberately not being tackled:
- Multi-VLAN routing
- Complex firewall rule sets
- High-availability or redundant networking
- Acting as the primary network gateway for a household

The most advanced networking use case envisaged at this stage is a **simple OPNsense firewall**, used for:
- Remote access
- Lab testing
- Temporary edge duties

For this project, networking should be good enough to support the services, flexible enough to move, and boring enough not to become the project all by itself.

---

## Constraints and known limits

This build is intentionally realistic about what a small, second-hand SFF system can and can’t do. Knowing the limits up front makes it easier to work *with* the hardware rather than constantly fighting it.

### Hard limits of the platform

Some constraints are simply baked in:

- **Form factor**  
  Being SFF limits physical expansion. Only low-profile PCIe cards are viable, and airflow will never rival a full tower.

- **Power delivery**  
  The stock PSU is designed for efficiency and stability, not high-wattage components. This rules out power-hungry GPUs and makes “serious” AI workloads a non-goal for this box.

- **Memory ceiling**  
  While up to 64 GB of RAM is supported, anything beyond the current configuration comes with a real cost and diminishing returns for this use case.

- **Single-host reality**  
  There is no redundancy. If this box is down, it’s down.

None of these are surprises — they’re part of the reason the system is quiet, cheap, and efficient.

### Accepted trade-offs

Some compromises are being made deliberately:

- **Efficiency over density**  
  Fewer, clearer workloads instead of packing everything into the smallest possible footprint.

- **Simplicity over cleverness**  
  Docker in a VM, straightforward networking, and boring defaults where possible.

- **Learning over optimisation**  
  Some resources will be “wasted” early on in exchange for clarity and flexibility.

These are conscious choices, not oversights.

### Possible (but not guaranteed) upgrades

The platform leaves room to grow, but nothing here is assumed or promised:

- Adding a **second NIC** (PCIe or USB) if network needs expand
- Installing a **low-profile GPU** for light AI inference or media offload
- Increasing **RAM** if workloads justify it
- Repurposing the HDD or replacing it entirely as storage needs evolve

All of these depend on cost, availability, and whether the use case actually materialises — not just because the slot exists.

In short, this system is designed to be useful *within its limits*.  
If those limits are hit, that’s a signal it’s time to scale out or build something new — not a failure of the original design.

---

## What’s intentionally deferred

To avoid accidental overengineering (and because this is meant to be a *starter* build), several things are being postponed on purpose.

### Features deliberately left for later

The following are consciously *not* part of the initial design:

- High availability, clustering, or failover
- Advanced networking (multiple VLANs, complex routing, fancy firewalling)
- GPU-heavy workloads or serious AI inference
- Full backup strategies with retention policies and off-site targets
- Any attempt to centralise “everything” onto this one box

None of these are bad ideas — they’re just not day-one requirements.

### Problems not being solved yet

This build is also choosing *not* to solve:
- Long-term data protection beyond basic snapshots
- Redundancy for power, storage, or networking
- Performance tuning beyond “this feels fine”
- Future-proofing for hypothetical workloads

If a problem doesn’t exist yet, it doesn’t get a solution yet.

### When this design should be revisited

Revisiting the design makes sense when one (or more) of the following happens:

- The system feels consistently constrained under normal use
- A service becomes important enough to justify guarantees
- Hardware limits are reached in a way that impacts day-to-day use
- The homelab grows beyond “one very capable node”

Until then, let the system do useful work, gather real experience, and resist the urge to redesign everything just because a spare evening exists (or at least try to resist).

In other words: build it, use it, learn from it — *then* change it.

---

> This design represents intent, not commitment.  
> Reality will be allowed to (and almost certainly will) disagree later.

