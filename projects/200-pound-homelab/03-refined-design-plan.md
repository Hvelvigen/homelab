# 03 – Refined Design Plan

This article reflects the **finalised design decisions** now that the hardware is in hand and key constraints are confirmed.
At this point, the design is no longer exploratory — it is the **intended implementation** for initial deployment.

The next stages after this document are physical setup, firmware configuration, and OS installation.

---

## Final hardware configuration

**Base system**
- Dell OptiPlex 5070 SFF
- Intel i5-9500 (6 cores)
- 16 GB DDR4 RAM
- 256 GB NVMe SSD
- 500 GB SATA HDD

**Power monitoring**
- Shelly Plug UK
  - Used to measure real idle and load power usage
  - Data informs always-on vs on-demand service decisions later

---

## Final networking layout

During refinement, it was decided that the device should be capable of acting as a **temporary household firewall/router**, rather than only a downstream lab node.

This was driven by three factors:

- The hardware is capable enough to handle basic routing and firewall duties
- Running OPNsense early provides real-world learning value
- A temporary edge role avoids committing to long-term network redesign

To support this, modest networking expansion was required.

Rather than investing in enterprise-grade NICs at this stage, two low-cost additions were sourced from eBay:

- A **PCIe Wi-Fi card** to allow the system to provide wireless connectivity directly
- A **USB Ethernet NIC** to provide a dedicated WAN interface

The combined cost for both items was **£12**, keeping the project aligned with its budget and ethos.

This approach deliberately favours **functional adequacy over elegance**.  
The goal is to enable real usage and experimentation without locking the system into a permanent edge role.

---

### Physical interfaces

| Interface | Role | Notes |
|---|---|---|
| Onboard Ethernet NIC | LAN | Primary internal network |
| USB Ethernet dongle | WAN | Upstream / ISP connection |
| PCIe Wi-Fi card | Wi-Fi AP | Wireless access via OPNsense |

This results in a **three-interface firewall/router** hosted entirely within the OptiPlex.

---

### Intended usage model

In this configuration, OPNsense is expected to:

- Act as the **default gateway** for the household network *temporarily*
- Provide:
  - Firewalling
  - NAT
  - DHCP
  - DNS forwarding
  - Basic Wi-Fi access
- Support:
  - Remote access experimentation
  - Firewall rule testing
  - “Real traffic” learning scenarios

This is explicitly **not** a permanent replacement for a dedicated router or access point setup.

The system may later be:
- Demoted back to a lab-only role
- Repositioned behind an existing router
- Reconfigured once additional hardware is available

---

### Accepted constraints and trade-offs

This design intentionally accepts the following limitations:

- **USB NIC for WAN**
  - Adequate for broadband speeds and lab use
  - Not suitable for sustained high-throughput routing
- **Wi-Fi via OPNsense**
  - Provides basic AP functionality only
  - No expectation of advanced roaming, band steering, or tuning
- **Single flat LAN**
  - No VLANs or segmentation initially
  - Keeps failure modes understandable during early use

If any of these constraints become problematic, the solution is to **change the role**, not to incrementally patch complexity onto this design.

---

This networking layout is considered **fit for purpose** for:
- A starter homelab
- Temporary household routing
- Learning-focused deployment

It is not intended to be clever, optimal, or future-proof — only *useful*.

---

## Host platform (locked)

- **Hypervisor:** Proxmox VE
- Installed directly onto the **256 GB NVMe**
- Host remains intentionally minimal:
  - No applications
  - No containers
  - No services

The Proxmox host exists to **run workloads**, not *be* one.

---

## Virtual machine layout (final)

### VM inventory (day one)

| VM | Purpose | Status |
|---|---|---|
| OPNsense | Firewall / Router / Wi-Fi | Required |
| Home Assistant | Home automation | Required |
| Ubuntu Server | Docker & services | Required |

No additional VMs are planned initially.

---

### Resource allocation

Allocations are sized to fit a **6-core CPU** with **intentional light oversubscription** to allow bust capacity where it matters.

| Component | vCPU | RAM | Notes |
|---|---|---|---|
| Proxmox host | 1 | 2–3 GB | Host responsiveness |
| OPNsense | 2 | 4 GB | Firewall + Wi-Fi |
| Home Assistant | 2 | 4 GB | Latency-sensitive |
| Ubuntu Server (Docker) | 2 | 6–8 GB | Shared services |

- Total allocated vCPU: **7**
- Light, intentional CPU overcommit (burst-oriented, not sustained)
- Some RAM intentionally left unallocated

Further CPU overcommit is a *future* optimisation, not a starting assumption.

---

## Container strategy (committed)

- All containers run **inside the Ubuntu Server VM**
- Docker is the only container runtime
- No LXC usage initially

### Rationale

- Clear separation between hypervisor and workloads
- Simple snapshot and rollback model
- Easy migration later
- Large support ecosystem

A small efficiency cost is traded for clarity and recoverability.

---

## Storage layout (final)

### NVMe (256 GB)
- Proxmox OS
- All VM disks
- Docker volumes (via Ubuntu VM)

### HDD (500 GB)
- Non-critical data
- Test files
- Temporary storage
- Future experimentation

This system is **not treated as a NAS**.

#### Rationale

This storage layout intentionally places the Proxmox OS, VM disks, and container data on a **single NVMe device**.

While this does not provide isolation between the host OS and workloads, it is a **deliberate trade-off** based on the constraints and goals of this build.

Key considerations:

- The system has **only one fast storage device** available
- Introducing manual partitioning would add rigidity and recovery risk without meaningful benefit
- Proxmox already provides logical separation between the host and VM disks via its storage backend
- This is a **starter homelab**, not an enterprise production platform

On a single physical disk, attempting to design for “safe OS reinstalls without backups” creates a **false sense of security**.  
If the NVMe device fails or is incorrectly reinitialised, both the OS and VM data are lost regardless of partitioning.

As a result, this design accepts that:

- Reinstalling Proxmox is an **exceptional event**, not a routine operation
- VM recovery is expected to occur via **backups or rebuilds**, not in-place preservation
- Simplicity and flexibility are prioritised over theoretical recoverability

When additional physical storage becomes available, VM disks can be moved off the NVMe to enable safer host reinstalls.  
Until then, this layout remains **fit for purpose**, easy to reason about, and aligned with the learning-focused goals of the project.

---

## Backup and resilience (explicitly limited)

At this stage:
- Snapshots: yes
- Backups: manual / basic
- Redundancy: none
- HA / clustering: out of scope

Failure modes are accepted as part of learning.

---

## Power and thermals (intent)

- Idle power target: **sub-15 W**
- Load observed via Shelly Plug
- Sustained thermal stress avoided

The box should remain:
- Quiet
- Cool
- Boring

If it stops being boring, something is wrong.

---

## What this design deliberately does NOT do

- No VLANs
- No NAS duties
- No serious AI workloads
- No production-grade edge routing
- No pretending USB networking is enterprise-grade

This is a **starter homelab**, not an enterprise edge appliance.

---

>This document reflects **decisions made**, not options considered.
>
>I expect the OptiPlex to disagree later.
