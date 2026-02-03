# Project: The £200 Challenge

---

## What’s this about?
Building a **starter homelab server for under £200** that’s *genuinely usable* from day one, doesn’t sound like it’s trying to achieve lift-off, doesn't pretend it's an Easy-Bake Oven, and still leaves sensible room to expand later.
  
The aim is to lower the barrier to entry without dumbing things down: make homelabbing **more approachable**, **financially realistic**, and **less intimidating** for people who are curious, capable, and maybe just not ready for racks, rails, and buyer’s remorse (the struggle is lived!).

---

## What brought this on?
Like many bad (good) ideas I've had (ask me about my OS), this one started on YouTube.

Creators like:
- Raid Owl — https://www.youtube.com/@RaidOwl  
- Hardware Haven — https://www.youtube.com/@HardwareHaven  
- Network Chuck — https://www.youtube.com/@NetworkChuck  
- Wolfgang — https://www.youtube.com/@WolfgangsChannel  
- Jeff Geerling — https://www.youtube.com/@JeffGeerling  

…all showed that you *don’t* need enterprise gear or racks to learn real skills and build genuinely useful systems.  

Most of that content, though, is very US-centric — so this project is about doing the same thing **properly, realistically, and cheaply in the UK**, using hardware you can actually find without selling a kidney.

---

## What are the requirements?
The server should:

- Run **Proxmox VE**
- Have a dedicated **OS disk** and a separate **data disk** (sda / sdb)
- Comfortably run **2–3 VMs** *plus* LXC or Docker without instantly pegging the CPU
- Support common “first homelab” services without drama

### Containers in scope
- Nginx Proxy Manager  
- Pi-hole  
- Nextcloud  
- Frigate  
- Minecraft server  
- Jellyfin  
- Uptime Kuma  
- Speedtest Tracker

### VM examples
- Linux
- Windows
- Purpose-built appliances (Home Assistant, OPNsense)

The idea isn’t to max things out immediately — there should be some breathing room left to grow.

---

## Device selection
### The struggle

When I first kicked this idea around, I was thinking of something closer to the £300–£400 mark and hadn’t fully internalised the concept of “if I don’t buy it, it doesn’t count towards the budget”.  
After a bit of feedback (and some self-reflection), I landed on **£200 as the hard ceiling** - yes, even with the price of RAM right now.

The actual searching didn’t take long.  
The *problem* was choice.

There are a *lot* of viable options once you start looking seriously, and it took several days of research to whittle that down to a sensible shortlist — and even longer to finally pick *the* box.

If you’re doing something similar, I’d strongly recommend digging into things like:
- Motherboard photos (to understand real expansion, not marketing expansion)
- Internal layout (airflow matters more than you think)
- PCIe slot placement and storage options

And you'll likely learn something halfway through - ask me what **VRM** is

None of this is difficult — it’s just fiddly.

The following devices were shortlisted because they are:
- Widely available second-hand in the UK
- Well supported by Linux/Proxmox
- Low power
- Expandable *enough* without becoming a NAS-in-disguise

| Device | Details | Pros | Cons |
|---|---|---|---|
| **Dell OptiPlex 5070** | 8th/9th-gen Intel, SFF, PCIe x16, NVMe + SATA, UHD 630 | Best overall balance; good expansion; handles i5-9500 comfortably; excellent Proxmox support; easy GPU/NIC adds | Slightly higher cost than entry models |
| **Dell OptiPlex 5060** | 8th-gen Intel only, SFF, PCIe x16, NVMe + SATA | Cheapest reliable 8th-gen option; low power; very common used | Less headroom than 5070; fewer board refinements |
| **Dell OptiPlex 7060** | 8th/9th-gen Intel, higher-end SFF, stronger power delivery | Best thermals/build of the Dell line; very stable | Often pricier; harder to justify on a tight budget |
| **HP EliteDesk 800 G4** | 8th-gen Intel, SFF, PCIe x16, NVMe + SATA | Solid Dell alternative; good firmware; reliable | Less common; tighter internal layout |
| **HP EliteDesk 800 G5** | 9th-gen Intel support, SFF, PCIe x16 | Comparable to OptiPlex 5070; good longevity | Usually more expensive than Dell equivalents |
| **Lenovo ThinkCentre M720s** | 8th-gen Intel, SFF, PCIe x16 | Quiet; stable; decent thermals | Fewer internal options; PSU/layout less flexible |
| **Lenovo ThinkCentre M920s** | 8th/9th-gen Intel, higher-end Lenovo SFF | Strong build quality; good expansion for SFF | Less common; pricing varies widely |

---

## Selected device
**Dell OptiPlex 5070 SFF**  
**i5-9500 · 16 GB RAM · 256 GB NVMe + 500 GB HDD**  
Purchased on eBay for **£175 (rounded up, postage included)**

### Why this one?
- Dell is a known quantity  
  (previous experience with a T640, R730xd, and an OptiPlex 3070)
- The 3070 was ruled out due to limited expansion and a quicker “base upgrade” ceiling
- The 5070 hits a sweet spot:
  - NVMe + up to **three SATA ports**
  - Up to **64 GB RAM**
  - Excellent Proxmox compatibility
  - **Low idle power (~9–12 W)**, ~50 W at peak
- **PCIe x16 (low-profile)** slot allows future NIC or GPU expansion
- Slightly better **power delivery** and thermal behaviour for sustained workloads (psst. here's where VRM is important)

In short: fewer compromises now, more expansion later.

---

## Why 8th gen instead of 9th gen?
Because the gains are **marginal for the cost**.

8th gen already brings:
- Six physical cores (i5)
- Strong single-thread performance
- Full Intel Quick Sync support

9th gen is nice, but for a budget-constrained build, the money is better spent on **RAM, storage, or future expansion**.

---

## Upgrade options
Planned (or at least plausible):

- **AI inference (lightweight)**  
  Limited by SFF power and thermals, but suitable for:
  - Frigate
  - Small LLM experiments
- **Multi-NIC expansion**  
  Possible future role as:
  - Firewall/router
  - Segmented lab edge
- **Storage upgrades**  
  Larger NVMe, SATA SSDs, or repurposing the HDD

The goal is that this box could absolutely be your final build — or it could be the gateway device that quietly drains your wallet and convinces you that “one more upgrade” is a perfectly reasonable idea. 

Homelabbing is flexible like that.

---
