# Uptime Kuma — Service Pattern  
*“Is it down?” becomes “I already know.”*

---

## Pattern Overview

**Pattern name:** Uptime Kuma  
**Host Service Layer:** Docker on Linux as a Service Host  
**Category:** Monitoring / Status & Health  
**Exposure:** Internal-only (optionally via reverse proxy)  
**Criticality:** Medium (high signal, low resource usage)  
**Intended users:** Homelab operator / anyone who wants a single place to see “is it up?”  

This pattern assumes the **Docker Host HSL** is already in place:

- `/srv/host-docker` on a dedicated data disk  
- `/srv/docker` and `/srv/logs` structured  
- Docker Root Dir moved away from `/var/lib/docker`  

Uptime Kuma becomes a first-class monitoring service.

---

## 1. Purpose

Uptime Kuma provides:

- HTTP/HTTPS/TCP/ICMP checks for services and devices  
- Simple uptime graphs and response time history  
- Built-in status pages for “is anything broken?” views  
- Notifications (email, Telegram, etc.) when something falls over  

This pattern ensures:

- Repeatable deployment on any Docker Host HSL  
- Clean paths for config and data  
- Predictable ports and DNS naming  
- Easy integration with a reverse proxy later  

---

## 2. When to Use / When Not to Use

### Use this pattern when:

- You want a **single pane of glass** for “up/down” across homelab services  
- You already have core infra online (OPNsense, Proxmox, Docker host)  
- You’d like a basic status page without building Grafana first  

### Probably skip (or defer) when:

- Nothing in the environment is “always on” yet  
- You’re already committed to another uptime tool and don’t want overlap  
- You’ll never set up notifications and only occasionally open the dashboard  

---

## 3. Prerequisites

We assume:

- Docker Host built according to the **Docker on Linux as a Service Host** pattern:
  - `/srv/host-docker`, `/srv/docker`, `/srv/logs` exist  
  - Docker Root Dir = `/srv/host-docker/runtime`
- You’re comfortable with:
  - `docker compose`
  - shell basics
- Networking is already sane:
  - Docker host has stable IP  
  - Basic LAN/VLAN connectivity proven

---

## 4. Architecture & Data Flow

High-level:

- One container: `louislam/uptime-kuma`  
- Listens on **port 3001** inside the container  
- Bound to a host port (e.g. `3001`) or to localhost only, then fronted by reverse proxy  
- All persistent data (DB, config, status pages) live under `/app/data` inside the container  

On this HSL:

- We store all app data under `/srv/docker/uptime-kuma/data`  
- Logs are Docker-managed initially; `/srv/logs/uptime-kuma` is reserved for future log shipping  

---

## 5. Service Definition

**Container:**

- **Image:** `louislam/uptime-kuma:2` (or `:latest` if you prefer)  
- **Container name:** `uptime-kuma`  
- **Restart policy:** `unless-stopped`  

**Ports:**

- Default: host `3001` → container `3001`  
- Later, when reverse proxy is in place, you can:
  - keep `3001` bound internally, or  
  - bind to `127.0.0.1:3001` and let NPM handle external access  

**Volumes:**

- `/srv/docker/uptime-kuma/data` → `/app/data`  

SQLite is used under the hood; **do not** put `/app/data` on NFS.

---

## 6. Directory Layout & Ownership

On the Docker host:

- **Service home:** `/srv/docker/uptime-kuma`  
- **Data:** `/srv/docker/uptime-kuma/data`  
- **Logs (reserved):** `/srv/logs/uptime-kuma`  

Create the directories:

    sudo mkdir -p /srv/docker/uptime-kuma/data
    sudo mkdir -p /srv/logs/uptime-kuma

If you want to be neat, align ownership with your usual service user (example: UID/GID 1000):

    sudo chown -R 1000:1000 /srv/docker/uptime-kuma
    sudo chown -R 1000:1000 /srv/logs/uptime-kuma

The image runs as its own user inside; host permissions just need to allow it to write to `/srv/docker/uptime-kuma/data`.

---

## 7. Deployment (Docker Compose)

**File:** `/srv/docker/uptime-kuma/docker-compose.yml`

Create the file:

    cd /srv/docker/uptime-kuma
    nano docker-compose.yml

Add:

    services:
      uptime-kuma:
        image: louislam/uptime-kuma:2
        container_name: uptime-kuma
        restart: unless-stopped

        environment:
          - TZ=Europe/London

        volumes:
          - /srv/docker/uptime-kuma/data:/app/data

        ports:
          - "3001:3001"
        # Later, if you only want localhost exposure:
        # ports:
        #   - "127.0.0.1:3001:3001"

Deploy:

    cd /srv/docker/uptime-kuma
    docker compose up -d

Quick sanity check:

    docker ps | grep uptime-kuma
    docker logs uptime-kuma --tail=50

If it starts without any obvious complaints, you’re ready for initial setup.

---

## 8. First Run & Basic Setup

Browse to:

    http://<docker-host-ip>:3001

On first load:

1. Create the initial admin account (email + password).  
2. Set **timezone** in the settings (match `TZ` for less confusion).  
3. Add your first few monitors, for example:
   - Proxmox GUI URL  
   - OPNsense WebUI  
   - UniFi Controller URL  
   - Speedtest Tracker URL  
   - IT-Tools URL  

Once those are green, you’ve got a minimal but useful monitoring view.

---

## 9. Logging & Troubleshooting

For now, logging is via Docker:

    docker logs -f uptime-kuma

Reserved path `/srv/logs/uptime-kuma` is for future use if you decide to:

- forward logs into a central log stack  
- keep structured logs per service in the filesystem  

Common things to check if it misbehaves:

- Port clashes (something else already on `3001`)  
- Volume path typos (`/srv/docker/uptime-kuma/data` vs `/srv/docker/uptime-kuma`)  

---

## 10. Updating

Standard HSL update flow:

    cd /srv/docker/uptime-kuma
    docker compose pull
    docker compose up -d
    docker logs uptime-kuma --tail=50

If logs are quiet and the UI loads, you’re done.

---

## 11. Backup & Restore

Back up at least:

    /srv/docker/uptime-kuma/data

That covers:

- monitors  
- status pages  
- settings  

**Restore pattern:**

1. Rebuild Docker Host HSL (if needed).  
2. Recreate `/srv/docker/uptime-kuma` and `docker-compose.yml`.  
3. Restore `data` directory into `/srv/docker/uptime-kuma/data`.  
4. `docker compose up -d` from `/srv/docker/uptime-kuma`.  

If the image tag is compatible, Uptime Kuma will come back with history intact.

---

## 12. Integration

Recommended next steps:

- Put Uptime Kuma behind your reverse proxy:
  - e.g. `https://status.lab.local` → `http://docker-host:3001`
- Add more monitors:
  - Docker host SSH/TCP checks  
  - DNS checks (Pi-hole / Unbound)  
  - External websites you care about  

Once this is in place, you’ve got a clean trio:

- **UniFi:** network edges and Wi-Fi  
- **Speedtest Tracker:** WAN performance over time  
- **Uptime Kuma:** “Is the thing actually up?” at a glance  

That’s a solid monitoring baseline for the project.

