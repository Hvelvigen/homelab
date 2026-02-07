# 04 – Pre-Power Checks & BIOS Decisions

This stage covers all actions taken **before first power-on** following hardware acquisition and refinement decisions.

The intent is to:
- Eliminate unknowns
- Reduce avoidable failure modes
- Start the software phase from a known-good baseline

No software is installed during this phase.

---

## Pre-power hardware preparation

Before powering the system for the first time, the OptiPlex was fully disassembled and reassembled.

This was done deliberately rather than “powering on to see what happens”.

### Actions performed

- Full external and internal clean
  - Dust removal from chassis, fans, and heatsinks
  - Visual inspection for damage or loose connectors or damaged capacitors
- CPU cooler removed and reseated
  - Existing thermal compound removed
  - New thermal paste applied
  - Cooler reinstalled with even mounting pressure
- Additional hardware installed
  - PCIe Wi-Fi card
  - USB Ethernet NIC (reserved for WAN use)
- Internal cabling checked and reseated
  - SATA
  - Power
  - Front-panel connections

### Rationale

Second-hand systems carry unknown history.

Performing a clean teardown and reassembly:
- Reduces thermal and stability issues later
- Ensures newly installed hardware is properly seated
- Avoids diagnosing “software problems” that are actually physical

This also establishes confidence that any later issues are **not due to neglect or residue from a previous life**.

---

## First power-on intent

The first power-on is expected to:

- POST cleanly
- Detect all installed hardware
- Enter firmware configuration without error

No operating system installation is attempted until BIOS configuration is verified.

---

## BIOS / firmware configuration

The BIOS is treated as part of the **platform foundation**, not a place for performance tuning or experimentation.

Before any settings were amended, the system firmware was verified to be the **latest available version** for the OptiPlex 5070.


Only settings with a clear impact on:
- Stability
- Virtualisation
- Predictability

are modified.

All other settings are left at defaults.

---

### Boot architecture: **UEFI only**

- Boot Mode: **UEFI**
- Legacy / CSM: **Disabled**
- Legacy Option ROMs: **Disabled**
- Attempt Legacy Boot: **Disabled**
- UEFI Boot Path Security: **Always, Except Internal HDD** (default retained)

**Rationale:**  
UEFI-only boot provides the cleanest and most predictable path for modern Linux hypervisors.  
Legacy boot paths were explicitly disabled to prevent fallback behaviour and reduce boot-time ambiguity.

No changes were made to the existing boot sequence entries; these are expected to be replaced automatically during Proxmox installation.

---

### Secure Boot: **Disabled**

- Secure Boot: **Disabled**
- Secure Boot Deployed Mode: **Not applicable / unchanged**
- Secure Boot keys: **Not initialised**

**Rationale:**  
Proxmox does not require Secure Boot, and disabling it avoids unnecessary friction during installation, kernel updates, and recovery.

This system is a homelab hypervisor, not a locked-down endpoint. Secure Boot can be revisited later if explicitly required.

---

### Storage configuration

- SATA Operation: **AHCI**
- RAID / Intel RST: **Disabled**
- SMART Reporting: **Enabled**

**Rationale:**  
AHCI exposes disks directly to the operating system, ensuring predictable behaviour, full SMART visibility, and compatibility with Linux storage stacks.

Firmware RAID (Intel RST) was disabled as it provides no benefit for this design and complicates Linux disk handling.

SMART was enabled to allow OS-level disk health monitoring.

---

### Integrated devices and peripherals

- Integrated NIC: **Enabled**
  - PXE: **Disabled**
  - UEFI Network Stack: **Disabled**
- USB Controllers: **Enabled**
- USB PowerShare: **Disabled**
- Audio: **Disabled**
- Serial Port: **Disabled**
- Wi-Fi / PCIe devices: **Enabled**
- Absolute / Computrace: **Disabled**

**Rationale:**  
Only hardware required for the intended workload is exposed to the OS.

PXE and pre-boot networking were disabled to avoid unnecessary boot complexity.  
USB PowerShare was disabled to reduce standby power draw and ensure predictable power states.  
Audio and serial ports were disabled as they are not required for a server-class workload.  
Absolute (endpoint tracking / persistence) was disabled as it provides no value in a homelab context and introduces opaque firmware-level behaviour.

---

### Platform security features

- TPM 2.0: **Enabled**
  - Attestation: **Enabled**
  - Key Storage: **Enabled**
  - SHA-256: **Enabled**
  - Clear / destructive actions: **Not bypassable**
- SMM Security Mitigation: **Enabled**
- Intel SGX: **Disabled**
- Intel Trusted Execution (TXT): **Disabled**

**Rationale:**  
TPM 2.0 was left enabled to preserve future compatibility (e.g. vTPM for guest VMs) without enforcing Secure Boot or measured boot today.

SMM Security Mitigation was retained to harden firmware execution paths at no cost.

SGX and Trusted Execution were disabled as they are not required for Proxmox or the intended workloads and add unnecessary complexity when Secure Boot is not in use.

---

### CPU configuration and power management

- Multi-Core Support: **All**
- Intel SpeedStep: **Enabled**
- Intel Speed Shift: **Enabled**
- CPU C-States: **Enabled**
- Intel Turbo Boost: **Enabled**
- CPU performance tuning: **Defaults retained**

**Rationale:**  
All physical cores are exposed to the OS to allow proper scheduling and VM density.

Modern CPU power-management features (SpeedStep, Speed Shift, C-States, Turbo Boost) were left enabled to balance low idle power with responsive burst performance.

No manual tuning was applied; stability and observed behaviour take precedence over theoretical optimisation.

---

### Virtualisation support: **Enabled**

- Intel Virtualization Technology (VT-x): **Enabled**
- VT for Direct I/O (VT-d / IOMMU): **Enabled**

**Rationale:**  
VT-x is required for hardware-accelerated virtual machines.  
VT-d enables proper device isolation and future PCIe passthrough if required.

Both features were enabled as foundational capabilities, even if not immediately exercised.

---

### Power behaviour

- AC Recovery: **Power On**

**Rationale:**  
The system may act as a temporary network gateway. Automatic power-on after AC loss ensures recovery without manual intervention.

---

## What is intentionally *not* configured in BIOS

The following areas were deliberately left untouched:

- Fan curves
- Overclocking or power-limit tuning
- Memory timing adjustments
- Wake-on-LAN tuning
- Vendor “performance” presets
- Manual boot entry manipulation

If tuning is required later, it will be driven by **observed behaviour and real constraints**, not assumptions.

---

## Exit criteria for this phase

This phase is considered complete when:

- The system POSTs reliably
- All installed storage is detected
- CPU virtualisation features are visible
- Added NICs appear in firmware
- No BIOS warnings or errors are present

Only once these conditions are met does the process move on to **Proxmox installation**.

---

This phase exists to ensure that any issues encountered later are **software problems**, not avoidable hardware or firmware mistakes.
