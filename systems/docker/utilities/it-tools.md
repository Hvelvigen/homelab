# IT-Tools — Service Pattern  
*A small-but-mighty internal toolbox for everyday technical tasks.*

---

## Pattern Overview

**Pattern name:** IT-Tools  
**Host Service Layer:** Docker on Linux as a Service Host  
**Category:** Utilities / Developer Tools  
**Exposure:** Internal-only (via reverse proxy)  
**Criticality:** Low–Medium  
**Intended users:** You, other technical humans, and anyone who has ever said “I just need to decode this one thing.”  

This pattern sits directly on top of the **Docker Host HSL**.  
If the host isn’t built according to that guide, or you don't have your own environment already configured, you may experience unexpected results…  
like the app running but the filesystem having an existential crisis.

---

## 1. Purpose

IT-Tools bundles dozens of handy utilities into a single tidy web interface — encoders, decoders, formatters, testers, hashers, prettifiers, and the occasional “oh thank goodness this is here” tool.

This Service Pattern ensures:

- IT-Tools runs cleanly, quietly, and **internally**  
- networking is controlled and predictable  
- updates don’t require detective work  
- you can explain its layout without sighing first  

Think of this pattern as giving IT-Tools a proper home rather than letting it squat on your host.

---

## 2. When to Use / When Not to Use

### 2.1 When to use this pattern
Use IT-Tools when you want:

- a fast, local alternative to random web utilities  
- a safe place to paste sensitive data *without* sending it to an ad network  
- a consistent, always-available toolbox for development and admin work  
- a zero-maintenance “nice to have” that becomes a “how did I live without this?”

### 2.2 When not to use this pattern
This pattern is **not** ideal if:

- you plan to expose IT-Tools directly to the internet (don’t… just don’t)  
- you need authentication or per-user isolation (IT-Tools is open to whoever can reach it)  

If you’re only testing the container for five minutes, you don’t need a pattern.  
This pattern is for *proper deployment* — when IT-Tools becomes a resident, not a visitor.

---

## 3. Assumptions & Prerequisites

We assume:

- You have a **Docker Host built with the Hvelvigen HSL**, meaning:
  - OS on `sda`, Docker on `sdb`
  - `/srv/host-docker`, `/srv/docker`, `/srv/logs` exist
  - Docker Root Dir = `/srv/host-docker/runtime`
- You know how to use:
  - `docker compose`
  - the shell
- You want IT-Tools reachable only via:
  - LAN,  
  - VPN,  
  - or the homelab’s grand traffic bouncer: the reverse proxy.

No credentials or secrets required — IT-Tools keeps life simple.

---

## 4. Architecture & Data Flow

A quick architectural sketch:

- IT-Tools is one lightweight container.
- It listens internally on port **80**.
- We bind it on the host to **127.0.0.1:8181**:
  - this keeps it private and stops the entire LAN from stumbling across it.
- If you run a reverse proxy, it will talk to IT-Tools on that port.
- If not, you’ll access it via SSH tunnelling or directly from the host.

Nothing clever — just clean, intentional traffic flow.

---

## 5. Directory Layout

Following Hvelvigen standards, IT-Tools lives in:

```
/srv/docker/it-tools/
  config/
  data/
  runtime/

/srv/logs/it-tools/
```

Create the directories:

```bash
sudo mkdir -p /srv/docker/it-tools/config
sudo mkdir -p /srv/docker/it-tools/data
sudo mkdir -p /srv/docker/it-tools/runtime
sudo mkdir -p /srv/logs/it-tools
sudo chown -R <your-uid>:<your-gid> /srv/docker/it-tools /srv/logs/it-tools
```

This structure means anyone who sees the service later will know exactly where everything lives — no scavenger hunts required.

---

## 6. Configuration Inputs

| Input            | Default              | Meaning                                                     |
|------------------|----------------------|-------------------------------------------------------------|
| Hostname / URL   | `it-tools.local`     | Reverse proxy friendly name                                 |
| Host bind port   | `127.0.0.1:8181`     | Keeps service internal-only                                 |
| Container port   | `80`                 | IT-Tools internal HTTP port                                 |
| PUID / PGID      | `1000 / 1000`        | User/group to run as                                        |
| TZ               | `Europe/London`      | Time zone                                                   |
| Network          | `ittools_net`        | Dedicated service network                                   |

Defaults are sane and quiet — ideal for small, local utility services.

---

## 7. Deployment (Docker Compose)

Create:

```bash
cd /srv/docker/it-tools
nano docker-compose.yml
```

Add:

```yaml
version: "3.9"

services:
  it-tools:
    image: ghcr.io/corentinth/it-tools:latest
    container_name: ittools
    restart: unless-stopped

    networks:
      - ittools_net

    # Internal-only; reverse proxy handles exposure.
    ports:
      - "127.0.0.1:8181:80"

    volumes:
      - ./config:/config
      - ./data:/data

    environment:
      PUID: "1000"
      PGID: "1000"
      TZ: "Europe/London"

networks:
  ittools_net:
    driver: bridge
```

Run it:

```bash
docker compose pull
docker compose up -d
```

Check:

```bash
docker ps | grep ittools
docker logs ittools --tail=50
```

If it starts without shouting, you're good.

---

## 8. Reverse Proxy Integration (Optional but Recommended)

If you use something like NGINX Proxy Manager:

- Target: `http://<docker-host>:8181`
- External URL: `https://it-tools.<your-domain>`

Benefits:

- TLS termination handled upstream  
- consistent access control  
- clean URLs  
- IT-Tools remains blissfully unaware of the outside world  

A perfect arrangement.

---

## 9. Operations

### 9.1 Start/Stop/Restart

```bash
docker compose up -d
docker compose down
docker compose restart
```

### 9.2 Logs

```bash
docker logs ittools --tail=100
```

Or, if using file logs:

```bash
ls /srv/logs/it-tools
tail -n 50 /srv/logs/it-tools/app.log
```

### 9.3 Backups

Backup these:

```
/srv/docker/it-tools/
/srv/logs/it-tools/
```

IT-Tools is mostly stateless, but back up anyway — future-you may want that config folder one day.

### 9.4 Updating

```bash
docker compose pull
docker compose up -d
docker logs ittools --tail=50
```

If it didn’t error, job done.

---

## 10. Decommissioning

When IT-Tools retires from service:

1. Stop and remove the container:
   *(From within the /it-tools folder)*
   ```bash
   docker compose down
   ```
3. Remove the network (optional):
   ```bash
   docker network rm ittools_net
   ```
4. Archive or delete its directories:
   ```bash
   sudo tar czf /srv/docker/it-tools-archive.tar.gz /srv/docker/it-tools
   sudo rm -rf /srv/docker/it-tools
   sudo rm -rf /srv/logs/it-tools
   ```
5. Remove or update your reverse proxy entry.

The goal: a clean exit, no leftovers haunting the host.

---

## 11. Validation Checklist

- [ ] `/srv/docker/it-tools` contains config, data, runtime  
- [ ] `/srv/logs/it-tools` exists  
- [ ] IT-Tools runs on **127.0.0.1:8181**  
- [ ] `ittools_net` Docker network exists  
- [ ] No unexpected containers named `it-tools` or similar  
- [ ] Reverse proxy routing works (if used)  
- [ ] Backups include service + logs  
- [ ] Service restarts cleanly with `docker compose up -d`

If you checked all those, congratulations — you’ve deployed IT-Tools correctly, neatly, and in true Hvelvigen fashion.  
It will now sit quietly in the corner until you need it… like a well-trained digital butler.

---
