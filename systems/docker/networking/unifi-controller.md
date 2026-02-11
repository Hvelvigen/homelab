# UniFi Network Application — Service Pattern
*A proper home for your UniFi Controller, not “that one random container that everything depends on”.*

---

## Pattern Overview

- **Pattern name:** UniFi Network Application  
- **Host Service Layer:** Docker on Linux as a Service Host  
- **Category:** Network Management  
- **Exposure:** Internal LAN (optionally via reverse proxy)  
- **Criticality:** Medium–High  
- **Intended users:** Homelab operator / network administrator  

This pattern assumes the Docker Host HSL is already in place:
- `/srv/host-docker` on dedicated disk  
- `/srv/docker` and `/srv/logs` structured  
- Docker Root Dir relocated from default  

The UniFi controller becomes a defined service, not an ad-hoc container.

---

## 1. Purpose

The UniFi Network Application manages:

- UniFi access points  
- UniFi switches  
- UniFi gateways  
- Wireless configuration  
- Guest networks and VLAN integration  
- Firmware lifecycle  

This pattern ensures:

- Predictable container deployment  
- Persistent and backed-up configuration  
- Controlled network exposure  
- Safe upgrade workflow  

---

## 2. When to Use / When Not to Use

### 2.1 Use this pattern when:

- You want a self-hosted UniFi controller  
- You standardise services on your Docker HSL  
- You require portability and recoverability  
- You manage multiple UniFi devices  

### 2.2 Do not use this pattern when:

- You rely on UniFi Cloud  
- You use a CloudKey or Dream Machine as controller  
- You only operate a single standalone AP  
- Your Docker host is disposable or experimental  

---

## 3. Assumptions & Prerequisites

- Docker Host HSL complete  
- `/srv` layout implemented  
- Docker Root Dir relocated  
- LAN routing functional  
- Controller not exposed directly to the public internet  

You are comfortable using:
- `docker compose`
- basic shell operations  
- container log inspection  

---

## 4. Architecture & Data Flow

- Single container deployment  
- Embedded MongoDB (no external database required)  
- Persistent data mapped to host  
- Host networking used for device discovery  

Host networking is used intentionally to avoid L2 adoption and discovery complications.

---

## 5. Directory Layout

- `/srv/docker/unifi-controller/config`
- `/srv/logs/unifi-controller`

Create directories:

```bash
sudo mkdir -p /srv/docker/unifi-controller/config
sudo mkdir -p /srv/logs/unifi-controller
sudo chown -R <uid>:<gid> /srv/docker/unifi-controller /srv/logs/unifi-controller
```

`config/` is the critical persistent data location.  
`logs/` is optional if file logging is required.

---

## 6. Configuration Inputs

| Input | Example | Purpose |
|-------|----------|---------|
| TZ | Europe/London | Log and scheduler timezone |
| PUID / PGID | 1000 / 1000 | Container runtime identity |
| MEM_LIMIT | 1024M | JVM memory hint |
| Network mode | host | Direct port binding |
| Data path | ./config | Persistent controller data |

---

## 7. Deployment (Docker Compose — Host Network)

- Navigate to service directory:

```bash
cd /srv/docker/unifi-controller
nano docker-compose.yml
```

- Compose file:

```yaml
version: "3.9"

services:
  unifi-controller:
    image: lscr.io/linuxserver/unifi-network-application:latest
    container_name: unifi-controller
    restart: unless-stopped
    network_mode: host

    environment:
      PUID: "1000"
      PGID: "1000"
      TZ: "Europe/London"
      MEM_LIMIT: "1024M"

    volumes:
      - ./config:/config
```

- Deploy:

```bash
docker compose pull
docker compose up -d
```

- Verify:

```bash
docker ps | grep unifi
docker logs unifi-controller --tail=80
```

---

## 8. Access

- Direct LAN access:

`https://<docker-host-ip>:8443/`

- Accept self-signed certificate warning on first access.

- Run initial setup wizard and adopt devices.

---

## 9. Operations

### Start / Stop

```bash
docker compose up -d
docker compose down
docker compose restart
```

### Logs

```bash
docker logs unifi-controller --tail=100
```

### Backup

- Primary backup path:

`/srv/docker/unifi-controller/config`

Example archive:

```bash
sudo tar czf /srv/docker/unifi-controller-backup-$(date +%F).tar.gz \
  -C /srv/docker/unifi-controller config
```

### Update

```bash
cd /srv/docker/unifi-controller
docker compose pull
docker compose up -d
```

Allow time for schema migrations after major version upgrades.

---

## 10. Decommissioning

```bash
docker compose down
sudo tar czf /srv/docker/unifi-controller-archive-$(date +%F).tar.gz \
  -C /srv/docker/unifi-controller config
sudo rm -rf /srv/docker/unifi-controller
sudo rm -rf /srv/logs/unifi-controller
```

Remove any reverse proxy or DNS references as required.

---

## 11. Validation Checklist

- Config directory exists and is owned correctly  
- Container running with host networking  
- UI accessible via LAN IP on 8443  
- Devices adopted and reporting connected  
- Backups tested  
- Upgrade workflow confirmed functional  

When these conditions are met, the UniFi Network Application is structured, reproducible, and maintainable within the Docker Host Service Layer.
