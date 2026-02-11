# Nextcloud All-in-One (AIO) — Service Pattern  
*The supported, opinionated, upstream way to run Nextcloud with minimal faff.*

---

## Pattern Overview

**Pattern name:** Nextcloud All-in-One (AIO)  
**Host Service Layer:** Docker on Linux as a Service Host  
**Category:** Files & Collaboration  
**Exposure:** HTTPS (443)  
**Criticality:** High (user data + primary cloud service)  
**Intended users:** Homelab operator + household users  

This pattern uses the **official Nextcloud AIO master container**, which then orchestrates:

- Nextcloud  
- Database  
- Redis  
- Collabora  
- Imaginary  
- Fulltext search (optional)  
- Talk (optional)  

All managed from a single AIO admin interface.

---

## Important Design Constraints

Nextcloud AIO expects:

- Ports **80 and 443** available on the host  
- Direct control of Docker (via mounted Docker socket)  
- No reverse proxy *in front* during initial bootstrap  

If you already have Nginx Proxy Manager on 80/443, you must:

- Temporarily stop it, or  
- Move NPM to another host, or  
- Use the advanced reverse proxy mode (not zero-touch)

For true zero-touch AIO:  
**Give it 80 and 443 directly.**

---

## Directory Layout

On the Docker host:

    /srv/docker/nextcloud-aio/
      mastercontainer/

    /srv/logs/nextcloud-aio/
      # reserved for future use

Create directories:

    sudo mkdir -p /srv/docker/nextcloud-aio/mastercontainer
    sudo mkdir -p /srv/logs/nextcloud-aio

Ownership can remain root; AIO handles internal permissions.

---

## Docker Compose (Official AIO Method)

File: `/srv/docker/nextcloud-aio/docker-compose.yml`

Create the file:

    cd /srv/docker/uptime-kuma
    nano docker-compose.yml

Add:

    services:
      nextcloud-aio-mastercontainer:
        image: nextcloud/all-in-one:latest
        container_name: nextcloud-aio-mastercontainer
        restart: unless-stopped

        ports:
          - "80:80"
          - "443:443"
          - "8080:8080"

        environment:
          - TZ=Europe/London

        volumes:
          - /srv/docker/nextcloud-aio/mastercontainer:/mnt/docker-aio-config
          - /var/run/docker.sock:/var/run/docker.sock:ro

Deploy:

    cd /srv/docker/nextcloud-aio
    docker compose pull
    docker compose up -d

Check:

    docker ps | grep nextcloud-aio

You should see the master container running.

---

## First Run (AIO WebUI)

Browse to:

    http://<docker-host-ip>:8080

You will be presented with:

- An initial password (displayed once in logs)

Retrieve it if needed:

    docker logs nextcloud-aio-mastercontainer

Copy the admin password shown.

Login to the AIO interface.

---

## Zero-Touch Guided Setup

Inside the AIO WebUI:

1. Set your domain (e.g. `cloud.lab.local` or real FQDN).  
2. Confirm ports 80/443 are reachable.  
3. Let AIO:
   - Pull required containers  
   - Provision DB  
   - Provision Redis  
   - Configure Collabora  
   - Generate TLS (self-signed or Let’s Encrypt)  

You click “Start containers” and it does the rest.

No manual DB config.  
No manual Redis config.  
No manual Collabora wiring.

---

## Resulting Containers (Managed by AIO)

After setup you will see multiple containers:

- nextcloud-aio-apache  
- nextcloud-aio-nextcloud  
- nextcloud-aio-database  
- nextcloud-aio-redis  
- nextcloud-aio-collabora  
- others depending on modules enabled  

**Do not manually edit or recreate these.**  
Manage everything through the AIO interface.

---

## Accessing Nextcloud

Once deployment finishes:

    https://cloud.lab.local

(or whatever domain you configured)

You will:

- Create the initial admin account in the browser  
- Log in  
- Start using files immediately  

Collabora is already wired internally.

---

## Reverse Proxy Considerations (Later)

If you want Nginx Proxy Manager later:

- Switch AIO to reverse proxy mode  
- Disable its built-in TLS  
- Proxy 443 to the AIO Apache container  

That is a second-phase change and not required for initial deployment.

---

## Backup Strategy

AIO stores persistent data under:

    /srv/docker/nextcloud-aio/mastercontainer

The actual Nextcloud data volume is managed internally by AIO.  
For proper backups:

- Use AIO’s built-in backup functionality  
- Or snapshot the Docker volumes at the host level  

Do not assume only the mastercontainer folder is sufficient for full restore.

---

## Update Strategy

Update the master container:

    cd /srv/docker/nextcloud-aio
    docker compose pull
    docker compose up -d

Then use the AIO WebUI to update internal containers.

---

## What This Gives You

- Officially supported deployment path  
- Collabora fully integrated  
- Automatic DB + Redis wiring  
- One admin UI for lifecycle management  
- Near zero manual configuration  

For your project, this replaces the custom multi-container compose approach and keeps the stack clean, supported, and future-proof.

