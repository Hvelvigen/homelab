# Hvelvigen Roadmap
A living outline of upcoming work, missing components, and planned refinements across the repository.

This roadmap keeps development transparent and helps me pick up exactly where I left off.

---

## 1. Core Documentation (Linux / Ubuntu)

### Completed
- Ubuntu Server installation (x64)
- Ubuntu Server post-installation baseline
- SSH configuration & hardening
- Disk management essentials
- Bash, terminal skills, networking, and Linux reference material

### In Progress
- Aligning cross-links between Linux guides
- Docker: IT-Tools recipe (initial)

### Planned
- `docker-on-linux-as-a-service-host.md`
- `it-tools-on-docker.md` (systems/docker version)
- Raspberry Pi installation guide
- Troubleshooting references:
  - SSH troubleshooting
  - Docker troubleshooting
  - Networking diagnostics

---

## 2. Recipes Framework

### Completed
- Initial recipe structure established
- IT-Tools recipe scaffolded

### Planned
- Full “Docker on Linux with Ubuntu” recipe chain:
  - Base OS → Post-install → SSH → Docker host → IT-Tools
- Future recipes (sample):
  - “Reverse Proxy with NGINX Proxy Manager”
  - “Monitoring Stack (Zabbix/Prometheus)”
  - “Home Assistant Deployment (generic)”
  - “Proxmox VE baseline + VM patterns”
  - “Windows Server baseline”
- Ensure all guides declare their role in recipe building

---

## 3. Systems Documentation

### Planned Systems Paths
These folders are declared in the root README but not yet populated:
- `systems/proxmox/`
- `systems/hyper-v/`
- `systems/windows/`
- `systems/docker/`
- `systems/home-assistant/`

### Near-term priorities
- Create at least one baseline stub per platform
- Add cross-linking from recipes to these new systems guides
- 
---

## 4. Templates & Copy/Paste Tools

### Completed
- Root structure defined (`templates/`, `docs/`, `systems/`)

### Planned
- “Copy/Paste → Go” folder for ready-use snippets
- HA & templates - Include Intuition (ONLY BASIC - don't add the propriatary stuff)
- Docker service templates (Portainer, Gitea, NPM, etc.)
- Windows PowerShell templates for homelab use - Use DV6 templates WITHOUT Modern Deployment

---

## 5. Documentation Philosophy & Accessibility

### Completed
- Lightweight documentation guide
- Structured guide style established (Repository Location, Purpose, Assumptions, etc.)

### Planned
- Add a “How to read these guides” beginner orientation
- Define a contributor stance (read-only, no external contributions)

---

## 6. Refactors & Quality Improvements

### Identified Improvements
- Reduce repetition across Bash / Terminal Basics / File Management (Or see Minor Risks)
- Audit cross-links for consistency across all Linux guides
- Ensure naming conventions are consistent in examples (`/srv/docker/...`)

---

## 7. Longer-Term Plans

### Future Expansions
- Home Assistant automation, dashboards, naming schema guides
- Network and VLAN design templates
- Full 'DV7 like' homelab build as a master recipe
- Optional: Video-aligned learning paths (Fancy, but YouTube? Like the PowerShell stuff?)
- Optional: Turn this repo into a browsable “static site” (Self-Hosted with B-WebApp-Logic)

---
