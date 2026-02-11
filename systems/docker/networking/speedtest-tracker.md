# Speedtest Tracker — Service Pattern  
*A proper, always-on WAN telemetry service — not “I’ll run a speedtest when it feels slow”.*

---

## Pattern Overview

- **Pattern name:** Speedtest Tracker  
- **Host Service Layer:** Docker on Linux as a Service Host  
- **Category:** Monitoring / Network Telemetry  
- **Exposure:** Internal-only (optionally via reverse proxy)  
- **Criticality:** Low–Medium (high diagnostic value)  
- **Intended users:** Homelab operator  

This pattern assumes the Docker Host HSL is already in place:

- `/srv/host-docker` on dedicated disk  
- `/srv/docker` and `/srv/logs` structured  
- Docker Root Dir relocated from default  

Speedtest Tracker becomes a defined service, not an ad-hoc container.

---

## 1. Purpose

Speedtest Tracker provides:

- Scheduled WAN speed tests (download, upload, latency)  
- Historical graphs and trend visibility  
- Failure tracking and uptime correlation  
- Evidence when discussing performance with your ISP  

This pattern ensures:

- Predictable container deployment  
- Persistent and backed-up configuration  
- Controlled exposure  
- Clean integration with reverse proxy and internal DNS  

---

## 2. When to Use / When Not to Use

### Use when:

- You want objective historical ISP data  
- You want to correlate outages with other services  
- You prefer scheduled telemetry over manual testing  

### Do not use when:

- You are on metered/mobile broadband  
- You only care about one-off manual speed tests  
- You will never review the historical graphs  

---

## 3. Prerequisites

- Docker Host deployed using the Docker Host HSL pattern  
- Docker Root Dir relocated to `/srv/host-docker/runtime`  
- Non-root service user configured (UID/GID known)  
- Optional reverse proxy (e.g., Nginx Proxy Manager)  
- Wired network connection recommended  

---

## 4. Service Design

### 4.1 Container

- **Image:** `lscr.io/linuxserver/speedtest-tracker:latest`  
- **Container name:** `speedtest-tracker`  
- **Restart policy:** `unless-stopped`  
- **Runtime user:** PUID/PGID (non-root)  

### 4.2 Data Paths

    /srv/docker/speedtest-tracker/
      config/

    /srv/logs/speedtest-tracker/
      # reserved for future log export

Bind mount:

    /srv/docker/speedtest-tracker/config → /config

SQLite database is sufficient for homelab use.

---

## 5. Environment Variables

Required:

- `PUID`
- `PGID`
- `TZ`
- `APP_KEY`
- `APP_URL`
- `DB_CONNECTION=sqlite`

Optional:

- `SPEEDTEST_SCHEDULE`
- `SPEEDTEST_SERVERS`
- `DISPLAY_TIMEZONE`
- `PRUNE_RESULTS_OLDER_THAN`

Generate APP_KEY:

    echo -n 'base64:'; openssl rand -base64 32

---

## 6. Directory Preparation

Create directories:

    sudo mkdir -p /srv/docker/speedtest-tracker/config
    sudo mkdir -p /srv/logs/speedtest-tracker

Set ownership (example UID/GID 1000):

    sudo chown -R 1000:1000 /srv/docker/speedtest-tracker
    sudo chown -R 1000:1000 /srv/logs/speedtest-tracker

---

## 7. Docker Compose

- Navigate to service directory:

 ```bash
cd /srv/docker/speedtest-tracker
sudo nano docker-compose.yml
```

- Compose file:
```yaml
    services:
      speedtest-tracker:
        image: lscr.io/linuxserver/speedtest-tracker:latest
        container_name: speedtest-tracker
        restart: unless-stopped

        environment:
          - PUID=1000
          - PGID=1000
          - TZ=Europe/London
          - APP_KEY=base64:YOUR_GENERATED_KEY_HERE
          - APP_URL=https://speedtest.lab.local
          - DB_CONNECTION=sqlite
          # - SPEEDTEST_SCHEDULE=0 * * * *
          # - SPEEDTEST_SERVERS=
          # - DISPLAY_TIMEZONE=Europe/London
          # - PRUNE_RESULTS_OLDER_THAN=365

        volumes:
          - /srv/docker/speedtest-tracker/config:/config

        ports:
          - "6875:80"
```
Deploy:

    cd /srv/docker/speedtest-tracker
    docker compose up -d

---

## 8. First Run

Access:

    http://<docker-host-ip>:6875

Immediately:

- Change default admin credentials  
- Confirm timezone  
- Set test schedule  
- Optional: configure server pinning  

---

## 9. Logging (Planned vs Current)

Planned:

- Dedicated log export into `/srv/logs/speedtest-tracker`

Current:

    docker logs -f speedtest-tracker

Docker logs are sufficient for now.

---

## 10. Updating

    cd /srv/docker/speedtest-tracker
    docker compose pull
    docker compose up -d

---

## 11. Backup

Include in backup set:

    /srv/docker/speedtest-tracker/config

Restoring this directory restores configuration and history.

---

## 12. Integration

- Reverse proxy: `speedtest.lab.local → 6875`
- Internal DNS record recommended
- Use alongside UniFi metrics for ISP vs LAN differentiation

---

Speedtest Tracker is lightweight, low risk, and high diagnostic value.  
It belongs on any structured Docker Host HSL.
