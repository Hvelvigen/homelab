# 06 – Secondary Storage and Early Housekeeping

With the host up and deliberately left alone, the next practical problem was storage.

The OptiPlex has a second internal SATA drive, and leaving it unused while I started piling services onto the NVMe would have been a strange kind of self-sabotage. 

This system is meant to get me through a **house move transition period** — roughly four weeks of “living between two places, painting walls, building furniture, and wanting something familiar to work”.

That means:
- some media,
- some documents,
- and a place for backups to exist locally without drama.

Not a NAS.  
Not a long-term archive.  
Just something useful.

---

## What this disk is (and isn’t)

This second disk is a **5¼″ internal SATA drive**, installed permanently in the chassis.

It is intended to be:

- bulk storage for non-latency-sensitive data  
- a landing zone for media and documents during the move  
- a local target for VM backups where possible  
- something I don’t feel bad about wiping or reshaping later  

It is *not* intended to be:

- a high-availability solution  
- a single source of truth  
- a replacement for proper backups  
- the beginning of a storage empire  

This distinction matters, because it keeps decisions grounded.

---

## Why storage comes *before* workloads

Before spinning up services like Nextcloud, Jellyfin, or anything media-adjacent, I wanted to answer a boring but important question:

> “Where does the data live, and what happens if I change my mind later?”

By dealing with the second disk early, I avoided:
- binding services to the wrong storage by accident,
- migrating data mid-move,
- or pretending that a rushed layout was a design.

This stage is about **giving the disk a job**, not locking it into a career.

---

## How I approached the setup

The approach here was intentionally conservative.

Rather than slicing the disk into rigid partitions to enforce a 20% backup / 80% storage split, I kept things flexible:

- a single filesystem,
- clearly named directories,
- documented intent rather than hard limits.

Roughly speaking:
- about **20%** of the disk is expected to be used for backups,
- the remaining **80%** for general storage (media, documents, odds and ends).

If those proportions drift during the move, that’s fine.  
Flexibility beats theoretical neatness here.

---

## Making it usable (without overengineering it)

The disk was:
- initialised cleanly,
- formatted once,
- mounted persistently,
- and made visible to the host in a predictable location.

The goal was that:
- it survives reboots,
- it’s obvious what it’s for,
- and it doesn’t require relearning six months from now.

Basic SMART visibility was left enabled so I can see if it starts complaining — not because I expect heroics from it, but because silence is easier to trust when it’s monitored.

---

## Windows access (and why it’s not solved here)

Before the transition period, some of the data on this disk needs to be reachable from a Windows device on the network.

This is a **convenience requirement**, not a platform one.

Crucially:
- I did **not** expose the Proxmox host itself to Windows file sharing
- I did **not** turn the hypervisor into a file server

Instead, the expectation is that:
- Ubuntu Server will be running,
- services that need access to this disk will live there,
- and any Windows access will be mediated through that layer.

Exactly *how* that’s done belongs with the workload documentation, not here.

This section is about deciding **where that responsibility lives**, not implementing it prematurely.

---

## Portability considerations

Although this is an internal disk, the system as a whole is being treated as **temporarily portable**.

During the transition period:
- the box should be able to come up on a new network,
- OPNsense can provide basic wireless access if needed,
- and familiar services (media, docs) should be reachable without redesign.

That’s why this disk is set up to support:
- Jellyfin or Plex,
- Nextcloud,
- and simple local backups.

All of that assumes the Ubuntu Server VM is running.  
If it isn’t, the data waits patiently.

That constraint is accepted up front, rather than worked around awkwardly.

---

## Other early housekeeping

This stage also covered a few small but worthwhile checks:

- confirming Proxmox sees the disk cleanly
- making sure it doesn’t collide with VM storage paths
- ensuring nothing “helpful” tries to auto-use it
- and generally removing uncertainty before workloads arrive

None of this is glamorous, but all of it reduces friction later.

---

## State at the end of this stage

At the end of this section, the system had:

- a second internal disk mounted and ready  
- a clear, documented purpose for that disk  
- no services yet bound to it  
- no assumptions about permanence  
- and no illusions about resilience  

It’s boring storage, doing a boring job, during a boring but necessary transition period.

Which is exactly what I need right now.

From here, the system is finally ready to host **actual services** —  with somewhere sensible for their data to live when they arrive.
