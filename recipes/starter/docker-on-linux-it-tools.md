# Recipe: Starter - Docker on Linux - IT-Tools

## 1. Purpose / Desired Outcome

By the end of this recipe, you will have:

- an Ubuntu Server host with a clean, security-conscious baseline
- Docker installed and configured as a service host
- a single IT-Tools container (service) available on your LAN
- a clear understanding of where your configuration and data live

This recipe is LAN-only and assumes **internal-only access** to IT-Tools behind your firewall.

---

## 2. Who This Recipe Is For

Suitable for beginners — no prior Docker experience needed.  
You should be comfortable with basic Linux terminal usage and SSH.

If you’re still building confidence in these areas, check the reference folder for support:

- **Understanding Bash on Ubuntu** — a brief guide to the shell you’ll be using
- **Linux Terminal Cheatsheet** — common commands, navigation, and patterns    
- **Getting Comfortable with SSH** — connecting safely and predictably to your server  
- **Working with Files & Directories** — essentials for editing, moving, and inspecting configs  
- **Basic Networking Essentials** — IP addresses, ports, localhost, and what they actually mean  

These references are lightweight, practical starting points — useful even if you’ve been around Linux for a while.

---

## 3. Prerequisites

Before starting, you should have:

- A **hypervisor or physical machine** ready to host Ubuntu (x64)  
  or a **Raspberry Pi** capable of running Ubuntu Server (ARM64).
- Basic understanding of:
  - SSH (connecting to a server from another machine)
  - copying and pasting commands into a terminal
- A working network with:
  - DHCP or a reserved/static IP for the server
- Time to complete the full flow without rushing:
  - expect **1–2 hours** end-to-end the *first* time you do this

No public DNS, reverse proxy, or SSL/TLS is required for this recipe.

---

## 4. Supported Hardware Profiles

This recipe supports two hardware paths:

### Profile A — x64 System (VM or Physical)

Typical scenarios:

- Proxmox / Hyper-V VM
- Small form factor PC or repurposed desktop

Use:

- **Guide:** `systems/linux/ubuntu/ubuntu-server-installation.md` (x64 path)
- System disk: 16–32 GB
- Data disk: sized according to your service needs (IT-Tools itself is lightweight but you may add more services later)

### Profile B — Raspberry Pi (ARM64)

Typical scenarios:

- Raspberry Pi 4 (or better) with SSD or reliable storage
  (SD cards work, but they’re not ideal long-term)

Use:

- **Guide:** `systems/linux/ubuntu/ubuntu-server-rpi-installation.md` (to be created)
- Prefer SSD over SD card for longevity
- Ensure adequate cooling and power

Both profiles converge on the same OS configuration, security baseline, and Docker setup.

---

## 5. Flow Overview

This recipe follows these phases:

1. **Base OS installation**  
   Install Ubuntu Server on your chosen hardware.

2. **OS post-install configuration**  
   Apply updates, baseline tools, time settings, and network checks.

3. **Access & security baseline**  
   Configure SSH safely, enable key-based authentication, and apply UFW and Fail2ban.

4. **Docker host profile**  
   Configure the system for Docker.  
   Install Docker and prepare a simple, maintainable folder structure.

5. **IT-Tools service pattern**  
   Deploy IT-Tools as a single container with a persistent configuration directory.

6. **Validation & documentation**  
   Confirm everything works and capture lightweight notes.

---

## 6. Step-By-Step Recipe Flow

Each step sends you to a guide.  
Come back here after each one to confirm what you should now have and note anything relevant for future-you.

### Path A — x64 Hardware
### 1. Install Ubuntu Server (x64)
Follow: `systems/linux/ubuntu/ubuntu-server-installation.md`

---

### Path B — Raspberry Pi (ARM64)
### 1. Install Ubuntu Server (RPi)
Follow: `systems/linux/ubuntu/ubuntu-server-rpi-installation.md`

---

After this point, both paths follow the same steps.

**You should now have:**
- a clean Ubuntu Server install on x64 hardware or a VM  
- stable host network connectivity  
- console login available  

**Document:**
- hostname  
- IP address (CIDR)  
- hypervisor/host details  
- Ubuntu version

---

### 2. Apply Ubuntu Server Post-Installation Configuration  
Follow: `systems/linux/ubuntu/ubuntu-server-post-installation.md`

**You should now have:**
- the system fully updated  
- baseline tools installed (e.g., btop)  
- UFW installed but not yet enabled or configured  
  (this is handled in the SSH baseline)

**Document:**
- any deviations from the guide (and please, feel free to share - continual improvement is bloody important and I'd appreciate the assist!)  
- anything notable with networking or DNS  
- any intentional changes you made during this phase

---

### 3. Configure SSH Access & Security  
Follow: `systems/linux/ubuntu/ssh-configuration.md`

**You should now have:**
- SSH installed and running on a non-default port (e.g. 2022)  
- key-based authentication fully working  
- password authentication disabled  
- root logins disabled over SSH  
- UFW enabled with only required ports open  
- Fail2ban monitoring SSH attempts  

**Document:**
- SSH port  
- UFW rule summary  
- location of SSH and Fail2ban configs  
- any safe-listed networks

---

### 4. Configure Docker Host on Ubuntu  
Follow: `systems/linux/docker/docker-on-linux-as-a-service-host.md`

**You should now have:**
- The system configured as a Docker host
- Docker installed and running  
- Docker Compose installed  
- a predictable directory layout for services  
  (e.g. `/srv/docker/it-tools/`)  

**Document:**
- Docker and Compose versions  
- Directory structure for container data  
- any Docker daemon settings you changed  

---

### 5. Deploy IT-Tools Service Pattern  
Follow: `systems/docker/utilities/it-tools.md`

**You should now have:**
- IT-Tools running on your LAN  
- a persistent directory for IT-Tools configuration  
- a clear URL for access  
  (`http://server-ip:port`)  

**Document:**
- container name  
- compose file location  
- host → container volume mappings  
- final access URL

---

### 6. Validation & Documentation  
Use the checkpoints above and:  
`docs/documentation/lightweight-documentation.md`

**You should now have:**
- a stable, secure Docker host  
- IT-Tools running reliably  
- your notes updated for future changes  

**Document:**
- final state  
- anything unusual  
- next steps you might want later  

---

## 7. Variants / Options (Optional)

This initial recipe keeps IT-Tools:

- **LAN-only**
- served directly by Docker on a non-privileged port
- without public exposure or reverse proxy

Future variants may include:

- fronting IT-Tools with a reverse proxy (NGINX Proxy Manager, Traefik, etc.)
- exposing it through HTTPS and a domain name
- integrating authentication via a central SSO solution

Those will be separate service patterns and recipes that build on the same Docker host baseline.

---

## 8. Validation / Final Outcome

At the end of this recipe, verify:

- You can SSH into the server on the configured SSH port using key-based authentication.
- UFW is enabled and only:
  - your SSH port  
  - your IT-Tools port  
  are open.
- The Docker service is running without errors.
- The IT-Tools container is:
  - listed by `docker ps` (as per the Docker guide)
  - reachable via `http://server-ip:port` from another device on the LAN.

If all of the above hold true, the recipe outcome is achieved.

---

## 9. Troubleshooting

If something isn’t working as expected, refer to:

- **SSH Troubleshooting**  
  - `systems/linux/ubuntu/ssh-troubleshooting.md` (to be created)

- **Docker & Container Troubleshooting**  
  - `systems/docker/docker-troubleshooting.md` (to be created)

- **Networking / Firewall Issues**  
  - any UFW or network troubleshooting notes under:
    - `reference/networking/` (to be created)

This recipe intentionally does **not** embed troubleshooting detail to keep it focused. Use the specialised guides instead.

---

## 10. What’s Next / Optional Extensions

Once you’re comfortable with this setup, potential next steps include:

- Adding a **reverse proxy** in front of IT-Tools
- Deploying additional containers alongside IT-Tools, using the same Docker host baseline
- Introducing **Portainer** or similar tooling for visual management
- Integrating logging and monitoring:
  - shipping logs to a central location
  - lightweight metrics for the host and containers
  - 
Future service patterns can reuse this host as a known-good, security-conscious baseline.
Just check the contradictions section of the service you’re interested in before you begin.
