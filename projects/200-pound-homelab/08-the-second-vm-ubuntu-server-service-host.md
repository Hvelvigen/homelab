# 08 – The Second VM: Ubuntu Server (Service Host)

With the firewall in place, the next requirement was obvious:  
something to **host services** so the rest of the lab could exist.

Before clients could connect, before access-layer behaviour could be tested, and before any further network decisions could be validated, there needed to be a reliable place to run infrastructure services — without polluting the hypervisor or overloading the firewall.

This VM exists to be useful, repeatable, and quietly boring.

It is not a pet system.  
It is not a general-purpose desktop.  
It is a **service host**.

At this stage, its responsibility is deliberately narrow:
- run Ubuntu Server cleanly,
- host Docker reliably,
- and provide a stable foundation for services the network will depend on next.

Specific applications — including the UniFi controller — are intentionally deferred to a later chapter.

---

## Why Ubuntu Server comes next

OPNsense establishes the network boundary.  
Ubuntu Server establishes the **service layer**.

Keeping those roles separate avoids blurring responsibilities between infrastructure and workloads.

Rather than running containers directly on the hypervisor, or embedding service logic into the firewall, the design stays simple:

- Proxmox runs VMs  
- OPNsense routes traffic  
- Ubuntu Server runs services  

This separation creates:
- clear fault boundaries
- easier backups and snapshots
- a predictable home for Docker workloads
- the ability to rebuild or move the service layer independently

In short: fewer surprises later.

---

## Role of this VM

This Ubuntu Server VM exists to act as:

- a dedicated Docker host
- a stable home for internal services
- the eventual platform for the UniFi controller

Nothing else.

If something needs to exist long-term, it belongs here — and it belongs in a container.

---

## Creating the Ubuntu Server VM

The Ubuntu Server VM follows the same philosophy as the rest of the build:  
**enough to be dependable, nothing more**.

Its purpose is straightforward. This system exists to become the service layer the rest of the lab depends on — specifically to host Docker cleanly and predictably — not to act as a tuning exercise or a density challenge. Once it is doing its job reliably, it should fade into the background.

To support that goal, the design deliberately avoids cleverness.

Key characteristics:

- modest, fixed CPU and memory allocation  
- a small, disposable OS disk  
- a separate data disk for services  
- no passthrough devices  
- no scheduler tweaks, pinning, or overcommit games  

The emphasis here is on **clarity and repeatability**.  
If this VM ever needs to be rebuilt, moved, or expanded, it should behave exactly as expected — without rediscovering early decisions that were optimised too soon.

Stability comes first.  
Optimisation, if it is ever needed, should be driven by observed behaviour rather than anticipation.

### VM specification

With that in mind, the service host VM was sized to comfortably support Docker workloads while remaining predictable and easy to reason about.

- **vCPU:** 2 (host CPU type)  
- **Memory:** 6 GB  
- **Disk (sda):** 32 GB — OS disk  
- **Disk (sdb):** 250 GB — data disk (`bulk-data`)  
- **Machine type:** q35  
- **BIOS:** OVMF (UEFI)  
- **Network:** connected to `vmbr1` (LAN behind OPNsense)

The OS disk is intentionally kept small and disposable.  
All service and container data is expected to live on the secondary disk, allowing the system to be rebuilt or replaced without entangling application state with the base operating system.

No passthrough hardware was assigned, and no scheduler or performance tuning was applied at this stage.

This VM exists to provide a stable, boring foundation for Docker-based services — nothing more, nothing less.

---

## Installing Ubuntu Server

The Ubuntu Server installer ISO was already present on the Proxmox host, having been uploaded earlier alongside the OPNsense image.

With the VM defined and storage attached, the ISO was mounted and the VM booted into the standard Ubuntu Server installer.

The installation itself followed the existing Ubuntu Server installation guidance available in `systems/linux/ubuntu`.

No customisations were introduced beyond what was required to get a clean, bootable system in place.

In practice, that meant:

- Ubuntu Server installed to the **32 GB system disk (sda)**
- the **250 GB data disk (sdb)** left untouched
- default package selection
- no additional roles or services enabled during install

The aim here was simply to establish a known-good base OS — not to pre-empt how it would later be used.

Once installation completed, the VM was rebooted and confirmed to boot cleanly from disk.

---

## Applying the Ubuntu baseline

With the base OS installed and booting cleanly, the next step was to bring the system up to a known, supportable baseline.

Rather than applying ad-hoc configuration, this followed the project’s existing Ubuntu guidance:

- `systems/linux/ubuntu/ssh-configuration.md`
- `systems/linux/ubuntu/ubuntu-server-post-installation.md`

These documents define the standard expectations for Ubuntu hosts in this project and are treated as **authoritative**.

At this stage, the intent was not to customise the system for a specific workload, but to ensure that:

- the OS is fully updated
- basic observability tools are in place
- logging behaviour is predictable
- SSH access is secure and reliable
- the host behaves consistently with other Ubuntu systems in the lab

No service-specific configuration was introduced during this phase.

---

## Documentation approach

Rather than duplicating command-by-command steps here, this journal records:

- that the standard Ubuntu post-install baseline was applied
- that SSH access was configured according to project guidance
- any intentional deviations, if they occurred

The detailed procedures live in the `systems/linux/ubuntu` guides and are referenced here to preserve a single source of truth.

This keeps the journal focused on **what was done and why**, while the guides remain focused on **how**.

---

## Preparing the host for Docker

With the Ubuntu baseline in place, the VM could be prepared as a **dedicated Docker service host**.

As with the OS configuration, this followed established project guidance rather than ad-hoc setup:

- `systems/linux/docker/docker-on-linux-as-a-service-host.md`

That guidance defines how Docker hosts are structured, where data lives, and how the system is expected to behave over time.

At this stage, the focus was on:

- preparing the secondary disk for service data
- establishing a predictable directory layout
- installing Docker from the official repository
- configuring Docker to keep runtime data off the OS disk
- ensuring the host behaves like an appliance, not a workstation

No application containers were deployed during this phase.

The goal was a Docker-capable platform that is:
- clean
- repeatable
- and unsurprising

---

## Network validation behind OPNsense

Before deploying any services, basic network behaviour was verified.

The intent here was not to test applications, but to confirm that the service host:

- receives an address from OPNsense
- can reach upstream networks when WAN is available
- resolves DNS correctly
- behaves like any other LAN client behind the firewall

This validation was intentionally simple.

If the host cannot obtain an IP, resolve names, or reach the network, then deploying containers would only obscure the real problem.

With these checks passing, the service host could be treated as a known-good participant on the internal network.

---

## State at the end of this stage

At the end of this section, the system consisted of:

- a running Ubuntu Server VM
- a hardened, consistent OS baseline
- secure SSH access
- Docker installed and correctly configured
- clean separation between OS and service data
- confirmed network connectivity through OPNsense
- no application containers deployed

Most importantly:
- the platform for services now exists and is trustworthy

The lab now has:
- a firewall
- a service host
- and a clear boundary between infrastructure and workloads

---

## What comes next

With the service host in place, the next step is to deploy **actual services**.

That work builds on everything established here:
- the network boundary
- the service layer
- the Docker host foundation

The first containers will introduce real behaviour, real traffic, and real dependencies.

Those belong in the next chapter.
