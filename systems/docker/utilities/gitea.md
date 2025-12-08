# Gitea — Service Pattern  
*A self-hosted Git platform that behaves, stays tidy, and doesn’t demand a PhD in DevOps to run.*

---

## Pattern Overview

**Pattern name:** Gitea  
**Host Service Layer:** Docker on Linux as a Service Host  
**Category:** Development Tools / Source Control  
**Exposure:** Internal-only (reverse proxy optional but recommended)  
**Criticality:** Medium–High for anyone doing real development  
**Intended Users:** You, collaborators, and anyone deserving read/write access to your code  

This pattern assumes you have a **Hvelvigen-compliant Docker Host**:

- `/srv/host-docker` mounted from `sdb`  
- `/srv/docker/*` for service data  
- Docker Root Dir correctly placed  
- UFW configured sensibly  
- No containers running directly outside service patterns  

If the host isn’t set up properly, Gitea will absolutely let you know — usually via a wall of errors that boil down to “where is my data folder?”

---

## 1. Purpose

Gitea provides a lightweight, GitHub-like platform where you can host repositories, track issues, manage pull requests, and collaborate on code.  
It’s small, fast, and doesn’t try to be everything to everyone — exactly the sort of tool that belongs in a well-kept homelab.

This Service Pattern ensures Gitea:

- lives in Hvelvigen’s directory structure  
- stores all persistent data cleanly under `/srv/docker/gitea`  
- exposes itself safely via reverse proxy  
- updates without drama  
- and remains portable for future migrations  

---

## 2. When to Use / When Not to Use

### 2.1 When to use this pattern

Use this pattern if:

- you want a **self-hosted Git platform** with minimal fuss  
- GitHub/GitLab is overkill or too public  
- you want clean integration with VS Code, CI runners, or homelab automation  
- you prefer your repositories to live within *your* walls  
- you’ll be storing code for apps, scripts, docs, or your game project

### 2.2 When *not* to use this pattern

You may want a different tool if:

- you need enterprise-level features from GitLab (heavy CI/CD, runners, etc.)  
- you require multi-tenant access with strict isolation  
- you plan to expose Gitea *directly* to the public internet (not recommended — use a proxy)

If you’re just testing Git concepts, a local folder works.  
If you’re building software as part of your homelab, **Gitea is perfect**.

---

## 3. Assumptions & Prerequisites

You need:

- A working Docker Host with:
  - `/srv/docker/gitea` created  
  - `/srv/logs/gitea` created  
- Ports available:
  - Internal Gitea port: `3000`  
  - Internal SSH port: `2022` (or `2222` mapped through Docker)  
- Optional reverse proxy (NGINX Proxy Manager, Traefik, Caddy)  
- A DNS entry for `gitea.<your-domain>`  

Your user must own the folders:

```bash
sudo chown -R <uid>:<gid> /srv/docker/gitea /srv/logs/gitea
```

Get your UID and GID:

```bash
id -u
id -g
```

---

## 4. Architecture & Data Flow

High-level overview:

- Gitea runs as **two containers**:
  1. **Gitea web service**  
  2. **Database** (MariaDB, SQLite, or PostgreSQL — this pattern uses **MariaDB** for simplicity)

- Gitea listens on:
  - `3000` for HTTP  
  - `2022` internally for Git-over-SSH  

- We bind the web UI internally to LAN access.  
- Reverse proxy handles HTTPS, public URL, and access control.  
- All data lives under `/srv/docker/gitea/{config,data,db}`.

---

## 5. Directory Layout

Structure:

```
/srv/docker/gitea/
  config/
  data/
  db/
  runtime/

/srv/logs/gitea/
```

Create:

```bash
sudo mkdir -p /srv/docker/gitea/config
sudo mkdir -p /srv/docker/gitea/data
sudo mkdir -p /srv/docker/gitea/db
sudo mkdir -p /srv/docker/gitea/runtime
sudo mkdir -p /srv/logs/gitea
sudo chown -R <uid>:<gid> /srv/docker/gitea /srv/logs/gitea
```

Everything Gitea needs lives here — backups become trivial.

---

## 6. Configuration Inputs

| Input              | Default                | Description                                  |
|-------------------|------------------------|----------------------------------------------|
| Web Port          | 3000                   | Internal Gitea HTTP port                     |
| SSH Port          | 2222                   | External SSH port → mapped to container 22   |
| Database          | MariaDB                | Reliable and small footprint                 |
| Network           | `gitea_net`            | Dedicated Docker network                      |
| Domain            | `gitea.<your-domain>`  | For reverse proxy                            |
| PUID/PGID         | 1000/1000              | User to run as                               |
| TZ                | Europe/London          | Time zone                                    |

---

## 7. Deployment — Docker Compose

Create:

```bash
cd /srv/docker/gitea
nano docker-compose.yml
```

Add:

```yaml
version: "3.9"

services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: unless-stopped
    environment:
      USER_UID: 1000
      USER_GID: 1000
      TZ: Europe/London
      GITEA__server__SSH_PORT: 2222
      GITEA__server__DOMAIN: gitea.local
      GITEA__server__ROOT_URL: http://gitea.local
    networks:
      - gitea_net

    ports:
      - "3000:3000"    # Change to 127.0.0.1:3000 if proxy-only
      - "2222:22"      # SSH for Git over SSH

    volumes:
      - ./config:/data/gitea
      - ./data:/data/gitea/repositories
      - ./runtime:/runtime
      - /srv/logs/gitea:/data/gitea/log

  mariadb:
    image: mariadb:10.6
    container_name: gitea-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: changeme
      MYSQL_DATABASE: gitea
      MYSQL_USER: gitea
      MYSQL_PASSWORD: gitea
    networks:
      - gitea_net
    volumes:
      - ./db:/var/lib/mysql

networks:
  gitea_net:
    driver: bridge
```

Start:

```bash
docker compose pull
docker compose up -d
```

Check:

```bash
docker ps | grep gitea
docker logs gitea --tail=50
```

Visit:

```
http://<docker-host-ip>:3000
```

Gitea will guide you through first-run setup.

---

## 8. Reverse Proxy Integration (Recommended)

If using NGINX Proxy Manager:

- **Domain:** `gitea.<your-domain>`
- **Forward Host:** `<docker-host-ip>`
- **Forward Ports:**  
  - HTTP → `3000`  
  - SSH (if you expose it) → `2222`
- **Scheme:** HTTP
- Enable:
  - Block Common Exploits
  - Websockets Support
- SSL:
  - Request Let's Encrypt certificate (internal or public)
  - Force SSL

Your public-facing URL becomes:

```
https://gitea.<your-domain>
```

SSH clone URL:

```
ssh://git@gitea.<your-domain>:2222/username/repo.git
```

It feels wonderfully official.

---

## 9. Operations

### 9.1 Start / Stop / Restart

```bash
docker compose up -d
docker compose down
docker compose restart
```

### 9.2 Logs

```bash
docker logs gitea --tail=100
docker logs gitea-db --tail=100
```

Or inside `/srv/logs/gitea/`.

### 9.3 Backups

Back up:

```
/srv/docker/gitea/config
/srv/docker/gitea/data
/srv/docker/gitea/db
/srv/logs/gitea
```

You now have **the entire Gitea instance**, including repositories and settings.

### 9.4 Updates

```bash
docker compose pull
docker compose up -d
```

Check logs afterward.

---

## 10. Decommissioning

To remove Gitea:

1.  
   ```bash
   docker compose down
   ```

2.  
   ```bash
   docker network rm gitea_net
   ```

3.  
   ```bash
   sudo tar czf gitea-backup.tar.gz /srv/docker/gitea
   sudo rm -rf /srv/docker/gitea
   sudo rm -rf /srv/logs/gitea
   ```

4. Remove reverse proxy entries.

Clean. Predictable. No ghosts.

---

## 11. Validation Checklist

Your Gitea deployment is Hvelvigen-compliant when:

- [ ] `/srv/docker/gitea/*` created and owned properly  
- [ ] `/srv/logs/gitea` exists  
- [ ] Gitea + MariaDB containers run cleanly  
- [ ] Gitea reachable at `http://host:3000` or via proxy  
- [ ] SSH works on `2222` if enabled  
- [ ] Backups include config + data + db  
- [ ] Compose restart works without error  
- [ ] No pet containers or stray repos outside the pattern  

If all boxes are ticked, congratulations —  
you now have a fully self-hosted Git platform running exactly the way it should.

Gitea will now sit quietly and professionally in your homelab, waiting to version-control your brilliance.

---
