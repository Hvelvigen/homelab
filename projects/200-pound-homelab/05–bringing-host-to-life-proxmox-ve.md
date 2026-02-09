# 05 – Bringing the Host to Life: Proxmox VE

This is the stage where the box finally stops being a collection of decisions and starts behaving like a system.

Up to this point, everything has been deliberate but inert:
- hardware chosen and checked,
- constraints accepted,
- firmware configured,
- nothing powered on “just to see”.

Installing Proxmox is the moment where all of that intent becomes real.

Still not exciting — but at least now it does something.

---

## The approach I took

For this build, I deliberately took the **least clever valid path** through the Proxmox installation.

There wasn’t a need for any advanced options for this deployment, and inventing one would have been enthusiasm masquerading as engineering.

So the choices were intentionally boring:

- Proxmox VE installed directly on the NVMe  
- Graphical installer  
- Default storage layout  
- No ZFS  
- No encryption  
- No RAID  
- No clustering assumptions  

This isn’t a production platform and it isn’t pretending to be one.  
The goal here is a stable, understandable hypervisor that can be rebuilt without drama if something goes sideways.

---

## Installing Proxmox

Rather than embedding a full installer walkthrough here, I followed my own Proxmox installation guide under:

    systems/proxmox/proxmox-ve-installation.md

What matters for this build is *how* I approached the install, not every screen I clicked through.

The installer itself is straightforward — the decisions around it are where people usually start negotiating with themselves.

---

## The things I paid attention to

Most of the installer screens are low-risk.  
A few aren’t.

These were the ones I treated as “stop and think” moments.

### Disk selection

This is the only genuinely destructive step.

The installer will wipe whatever disk you point it at, without ceremony or follow-up questions.  
Because the OptiPlex has more than one drive installed, I double-checked this before continuing.

If you’re ever unsure at this step, the correct move is to stop — not to guess and hope future-you understands.

---

### Hostname

I chose the hostname deliberately and treated it as final.

This name becomes:
- the Proxmox node name,
- part of the web UI identity,
- and something you’ll see constantly in logs and menus.

It *can* be changed later, but doing so is just friction I didn’t need to introduce to my own life.

---

### Networking

I kept networking intentionally simple.

For this build, that meant:
- a single active interface,
- DHCP with a reservation upstream,
- no VLANs,
- no bonding,
- no cleverness.

The only requirement at this stage was that the web UI loaded reliably from another machine.

Anything more than that would have been solving problems I didn’t yet have — or worse, creating new ones.

---

### Time zone

This is a small thing that causes disproportionately annoying issues when it’s wrong, so I set it accurately.

Logs, certificates, and timestamps all read much more convincingly when time itself agrees with you.

---

## First boot and initial sanity check

After installation and reboot, the system came up cleanly:

- no firmware complaints,
- no missing hardware,
- management IP displayed on the console,
- web UI reachable immediately.

At that point, Proxmox existed — but it wasn’t yet something I was willing to build on.

Before creating any VMs, I wanted to update the host and remove any remaining “fresh install optimism”.

---

## Post-install baseline

From here, I switched to the post-install Proxmox guide under:

    systems/proxmox/proxmox-ve-post-installation.md

That’s where I handled:
- repository configuration (no-subscription),
- full system updates,
- basic tooling,
- time synchronisation checks,
- and setting up a non-root administrative user.

None of that is unique to this build, which is exactly why it lives outside the £200 series.

What *is* specific to this build is the decision to stop once that baseline was complete — and to resist the temptation to keep poking at it.

---

## Where I deliberately stopped

Once:
- the host was fully updated,
- repositories were correct,
- the web UI was quiet,
- and no warnings were present,

I stopped changing things.

No extra packages.  
No tuning.  
No “just in case” tweaks.

From this point on, the Proxmox host exists purely to **run workloads**, not to be experimented on itself.

This boundary is intentional and, historically speaking, easy to ignore.

---

## State at the end of this stage

At the end of this section, the system was:

- powered on and stable (averaging ~12 W idle)  
- running Proxmox VE cleanly  
- reachable on the network  
- fully updated  
- hosting **nothing**  

Which might not sound like much — but it’s exactly where I wanted it.

From here on, the work moves *above* the hypervisor:
- virtual machines,
- services,
- and the things that actually justify the box being powered on.

That’s where the interesting decisions start — and where mistakes are much easier to undo.
