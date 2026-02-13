# 07 – The First VM: OPNsense

If nothing can talk to anything else, everything else is academic.

The first VM in this build was always going to be the firewall. 

Not because it’s exciting, but because **every other decision depends on it working**. Storage, services, media, even basic convenience during a house move all fall apart very quickly if the network isn’t there.

So before dashboards, containers, or anything that looks impressive, I built the most boringly important VM in the stack: **OPNsense**.

---

## Why OPNsense comes first

This system is being treated as a **temporary, portable homelab** during a house move. For a few weeks, it needs to:

- power on anywhere
- provide wired and wireless access
- tolerate questionable upstream connectivity
- behave consistently even when the environment doesn’t

That immediately rules out relying on whatever router happens to be nearby.

OPNsense gives me:
- a predictable network boundary
- DHCP and DNS I control
- VPN access when needed
- the ability to survive on tethered mobile data if fixed WAN isn’t available

It doesn’t need to be clever. It just needs to work.

---

## The (deliberately imperfect) network model

This setup makes a compromise up front, and it’s worth stating it clearly.

### Proxmox management NIC as WAN

The **onboard NIC** on the Proxmox host serves two roles:

- Proxmox management
- WAN interface for OPNsense

Is this best practice?  
No.

Is it acceptable *here*?  
Yes — because this is:
- temporary
- trusted
- single-node
- and not pretending to be a hardened edge deployment

This trade-off removes complexity during a transition period and keeps the system usable in unpredictable environments.

---

### LAN and Wi-Fi separation

On the LAN side, things are cleaner:

- **USB Ethernet NIC**
  - Connected to a UniFi 5-port 2.5 GbE switch
  - Acts as the wired LAN behind OPNsense

- **PCIe Wi-Fi card**
  - Passed through directly to the OPNsense VM
  - Used purely as a wireless access point

The Proxmox host itself stays wired and boring.  
Wi-Fi belongs to the firewall.

---

## Creating the OPNsense VM

With the design settled, the VM itself was straightforward.

### Uploading the OPNsense installer ISO

Before creating the VM, the OPNsense installer ISO needs to be available to Proxmox.

I downloaded the OPNsense ISO and uploaded it directly to the Proxmox host:

- Navigate to **Datacenter → <Node>**
- Select the storage backing the Proxmox host (typically `local`)
- Go to **ISO Images**
- Use the **Download from URL" to import the OPNsense installer ISO
  - It's worth doing the Checksum validation and Proxmox makes it easy for you to do this here

This makes the installer available so the VM can boot cleanly without involving external media or temporary mounts.

Once the ISO is present, Proxmox can treat it like any other virtual install media.
Whilst here (and because I knew I'd need it later) I also added the **Ubuntu Server 24.04.3** ISO and lamented the lack of Home Assistant ISO.

### VM specification

This was intentionally modest.

- **vCPU:** 2 (host CPU type)  
- **Memory:** 4 GB  
- **Disk:** 32 GB (thin provisioned)  
- **Machine type:** q35  
- **BIOS:** OVMF (UEFI)

For a firewall VM, anything beyond this would have been speculative rather than justified.

I enabled **Start at boot** so the network stack comes up automatically with the host, and added a few simple tags — *BSD*, *OPNsense*, *Firewall* — purely to make the VM easy to spot later when the list gets longer.

The OPNsense installer ISO was attached from local storage, and I left the guest OS profile as-is. Accuracy here doesn’t buy anything, and the Proxmox defaults are the least surprising option.

Networking was attached to `vmbr0` for WAN, reflecting the earlier decision to let the Proxmox management NIC double as upstream connectivity in this temporary setup.
With the WAN side in place, I then created a second bridge, `vmbr1`, to represent the internal LAN.

This bridge is backed by the USB Ethernet NIC and exists solely to provide a shared network segment for:
- the OPNsense LAN interface,
- and any other VMs that need to live behind the firewall.

To add the new "LAN" connection to the VM:
- create a new Linux bridge (`vmbr1`) on the node,
- attach the USB NIC as the bridge port,
- leave IP configuration unset on the host.

Proxmox’s role here is limited to simple frame forwarding.  
All addressing, routing, and policy decisions remain the responsibility of OPNsense.

The VM disk lives on the **default Proxmox VM storage** (the NVMe-backed `local-lvm`), not the secondary HDD — deliberately.

OPNsense itself:
- has minimal disk requirements,
- benefits from fast, low-latency storage,
- and doesn’t generate large or growing datasets in this configuration.

Logs, configuration, and state remain small and predictable. There’s no practical benefit in pushing the firewall onto bulk storage just to keep things visually tidy.

No ballooning, no overcommit heroics — just enough resource to do the job reliably and then get out of the way.

---

### Network interfaces

Once the VM was created, I attached **multiple network interfaces**, each with a very specific and intentionally constrained role.

1. **WAN**
   - Connected to `vmbr0`
   - This bridge maps to the Proxmox management NIC
   - *Attached during VM creation*

   This provides upstream connectivity with minimal moving parts. It’s not a textbook edge design, but it’s a conscious and documented compromise for a temporary, portable setup.
   
3. **LAN (wired)**
   - Provided via a USB Ethernet NIC
   - The USB NIC is owned by Proxmox and backs a dedicated bridge (`vmbr1`)
   - The OPNsense LAN interface is a virtual NIC attached to `vmbr1`

   This allows other VMs on the host to attach to the same LAN segment, communicate with each other, and route traffic through OPNsense as intended.

   Proxmox remains responsible only for bridging.  
   OPNsense remains responsible for DHCP, DNS, and routing.

   The USB NIC itself uses a Realtek chipset, which is well supported in this role and behaves predictably when used as a bridge-backed interface.

4. **LAN (wireless)**
   - Provided via a PCIe Wi-Fi card
   - Passed through directly to the OPNsense VM as a PCI device
   - Configured with **PCI-Express** and **All Functions** enabled

   This gives OPNsense full ownership of the wireless hardware, allowing it to behave like a proper access point rather than a host-mediated adapter.

   For passthrough devices like this, selection by **vendor/product ID** is preferred over bus paths to avoid surprises across reboots.

Wired LAN and Wi-Fi are treated symmetrically:
- both are owned entirely by OPNsense,
- neither is bridged through the host,
- and neither requires host-level network configuration.

The Proxmox host remains wired and deliberately boring.  
The firewall does the networking.

---

## USB phone tethering (because reality exists)

One of the explicit requirements for this setup is the ability to function **without fixed WAN**.

For that reason, I planned for **USB phone tethering** as a fallback WAN option.

The approach is simple:

- Plug the phone into the Proxmox host via USB
- Pass the USB device directly through to the OPNsense VM
- Let OPNsense treat it as an alternate WAN interface

This keeps:
- Proxmox unaware of mobile networking
- routing, NAT, and failover logic inside the firewall
- the host clean and predictable

Is it elegant?  
No.

Is it incredibly useful during a move?  
Yes.

That’s the bar.

---

## Installing OPNsense

With the VM created and hardware attached, I mounted the OPNsense installer ISO and booted the VM.

The install itself is uneventful:
- boot
- install to disk
- reboot
- initial console menu

Review the systems/bsd-opnsense guidance for the how to of it all.

At this stage, the goal was not fine-tuning or feature enablement — just getting the OS installed and booting cleanly.

Once installed:
- WAN and LAN interfaces were assigned
- a LAN interface was present
- basic services were available in their default state

That’s enough to stop.

No attempt was made to fully validate connectivity at this stage.

---

## What I didn’t configure yet (on purpose)

At this point, OPNsense was installed and running — and I left it there.

I did **not** yet configure:
- advanced firewall rules
- VPN specifics
- Unbound tuning
- DNS overrides
- traffic shaping
- multi-WAN logic
- final LAN or WLAN design
- end-to-end connectivity testing

Those are important, but they deserve their own attention and the means to test.

This stage is about **bringing the firewall into existence**, not integrating it into the network.

---

## Notes on intentional deferral

LAN and WLAN configuration were deliberately left incomplete.

No client devices were connected and no routing or access-layer validation was performed.

This is intentional.

The next dependency is an Ubuntu Server VM hosting the UniFi controller.  
Network validation will be done only once:
- the controller is online
- the UniFi device is adopted
- a client device can be connected through UniFi

This avoids making early assumptions about subnet layout or access-layer behaviour and avoids reworking the network twice.

---

## Interim console changes

Only minimal changes were made via the console where required.

Specifically:
- the LAN interface IP was changed from the default `192.168.x.x` range to `10.22.x.x/x`

No other configuration, tuning, or validation was performed.

---

## State at the end of this stage

At the end of this section, the system consisted of:

- a running OPNsense VM
- interfaces defined but not yet exercised
- default services in place
- no connected clients
- no validated routing or access paths

Most importantly:
- the firewall now exists as a platform dependency

Everything else will build on this once the service layer is in place.

For now, the firewall is intentionally boring, minimally touched, and waiting — which is exactly what it should be at this stage.

