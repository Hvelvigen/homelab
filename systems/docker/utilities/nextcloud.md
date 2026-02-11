# Nextcloud + Collabora — Service Pattern  
*Files, sync, and in-browser editing, all as a first-class Docker service.*

---

## Pattern Overview

**Pattern name:** Nextcloud + Collabora  
**Host Service Layer:** Docker on Linux as a Service Host  
**Category:** Files & Collaboration  
**Exposure:** Internal-only (fronted by reverse proxy)  
**Criticality:** Medium–High (user data lives here)  
**Intended users:** Homelab operator + household users  

Assumes the **Docker Host HSL** is already in place:

- `/srv/host-docker` mounted on dedicated data disk  
- `/srv/docker` and `/srv/logs` structured  
- Docker Root Dir relocated from `/var/lib/docker`  

Nextcloud is treated as a **defined service** with its own DB, cache, and office suite (Collabora), not a one-off container.

---

## 1. Purpose

This pattern delivers a “batteries included” Nextcloud stack:

- Nextcloud (files, sharing, sync, apps)
- MariaDB (database)
- Redis (local cache, locking)
- Collabora CODE (browser-based document editing)

Goals:

- Minimal manual configuration:
  - Stack is fully wired via `docker-compose.yml`
  - Database and Redis pre-configured
  - Admin account created on first start
- Post-deploy interaction happens **only in the Nextcloud WebUI**:
  - Finish initial admin login
  - Install/enable “Collabora Online” app
  - Point it at your Collabora URL

---

## 2. Host Assumptions

- Docker Host HSL already built and maintained
- Reverse proxy (e.g. Nginx Proxy Manager) will eventually front:
  - `cloud.lab.local` → Nextcloud
  - `office.lab.local` → Collabora
- Internal DNS can resolve those names to the Docker host / reverse proxy

You can start without the reverse proxy by talking directly to host ports, then drop NPM in later.

---

## 3. Architecture

Services in this stack:

- `nextcloud-app`  
  - Image: `nextcloud:apache`  
  - Depends on `nextcloud-db`, `nextcloud-redis`  
  - Uses env vars to auto-install and configure DB/cache/admin

- `nextcloud-db`  
  - Image: `mariadb:11` (or current stable)  
  - Exposed only on the internal Docker network

- `nextcloud-redis`  
  - Image: `redis:alpine`  
  - Used for file locking and caching; internal-only

- `collabora`  
  - Image: `collabora/code`  
  - Allows connections only from the Nextcloud hostname via `domain` regex

All containers share a dedicated Docker network: `nextcloud-net`.

---

## 4. Directory Layout

On the Docker host:

    /srv/docker/nextcloud/
      app/          # Nextcloud files, config, data (excluding external storage)
      db/           # MariaDB data directory
      redis/        # Redis data (optional persistence)
      collabora/    # reserved for future config/customisation

    /srv/logs/nextcloud/
      app/          # reserved for future application logs
      collabora/    # reserved for future collabora logs

Create directories:

    sudo mkdir -p /srv/docker/nextcloud/app
    sudo mkdir -p /srv/docker/nextcloud/db
    sudo mkdir -p /srv/docker/nextcloud/redis
    sudo mkdir -p /srv/docker/nextcloud/collabora

    sudo mkdir -p /srv/logs/nextcloud/app
    sudo mkdir -p /srv/logs/nextcloud/collabora

Ownership:

- The official images handle UID/GID internally.
- Host directories just need to be writable by Docker; if in doubt:

    sudo chown -R root:root /srv/docker/nextcloud
    sudo chown -R root:root /srv/logs/nextcloud

(You can tighten this later if you standardise on a service UID/GID.)

---

## 5. Environment Decisions

Baseline values (adjust to taste):

- Nextcloud hostname: `cloud.lab.local`
- Collabora hostname: `office.lab.local`
- Timezone: `Europe/London`
- Nextcloud admin:
  - User: `ncadmin`
  - Password: **set a strong one**

Database:

- DB name: `nextcloud`
- DB user: `nextcloud`
- DB password: strong unique password

Collabora:

- Admin console creds (optional but recommended):
  - `username`: `admin`
  - `password`: strong password
- `domain` must be a **regex** for the Nextcloud host – e.g. `cloud\\.lab\\.local`

---

## 6. Docker Compose

**File:** `/srv/docker/nextcloud/docker-compose.yml`

    version: "3.8"

    services:
      nextcloud-db:
        image: mariadb:11
        container_name: nextcloud-db
        restart: unless-stopped
        command: >
          mysqld
          --transaction-isolation=READ-COMMITTED
          --binlog-format=ROW
        environment:
          - MYSQL_ROOT_PASSWORD=CHANGE_ME_ROOT
          - MYSQL_DATABASE=nextcloud
          - MYSQL_USER=nextcloud
          - MYSQL_PASSWORD=CHANGE_ME_DB
        volumes:
          - /srv/docker/nextcloud/db:/var/lib/mysql
        networks:
          - nextcloud-net

      nextcloud-redis:
        image: redis:alpine
        container_name: nextcloud-redis
        restart: unless-stopped
        # For homelab, Redis is only on the internal network, no auth.
        # If you want auth, add requirepass and wire REDIS_HOST_PASSWORD in Nextcloud.
        command: ["redis-server", "--save", "", "--appendonly", "no"]
        volumes:
          - /srv/docker/nextcloud/redis:/data
        networks:
          - nextcloud-net

      nextcloud-app:
        image: nextcloud:apache
        container_name: nextcloud-app
        restart: unless-stopped
        depends_on:
          - nextcloud-db
          - nextcloud-redis
        environment:
          # Basic env
          - TZ=Europe/London

          # Auto-setup admin user (first run only)
          - NEXTCLOUD_ADMIN_USER=ncadmin
          - NEXTCLOUD_ADMIN_PASSWORD=CHANGE_ME_ADMIN

          # Database config
          - MYSQL_HOST=nextcloud-db
          - MYSQL_DATABASE=nextcloud
          - MYSQL_USER=nextcloud
          - MYSQL_PASSWORD=CHANGE_ME_DB

          # Redis cache / locking
          - REDIS_HOST=nextcloud-redis

          # Trusted domains and overwrite settings for reverse proxy
          - NEXTCLOUD_TRUSTED_DOMAINS=cloud.lab.local
          - NEXTCLOUD_OVERWRITEHOST=cloud.lab.local
          - NEXTCLOUD_OVERWRITEPROTOCOL=https
          - NEXTCLOUD_OVERWRITECLIURL=https://cloud.lab.local

          # Optional PHP limits (sensible homelab defaults)
          - PHP_MEMORY_LIMIT=1024M
          - PHP_UPLOAD_LIMIT=1024M

        volumes:
          - /srv/docker/nextcloud/app:/var/www/html

        # Direct exposure for initial use; later put behind Nginx Proxy Manager
        ports:
          - "8081:80"

        networks:
          - nextcloud-net

      collabora:
        image: collabora/code
        container_name: collabora
        restart: unless-stopped
        # Collabora needs to see the real hostname Nextcloud will use.
        environment:
          - TZ=Europe/London
          # Regex for Nextcloud host. Escape dots.
          - domain=cloud\\.lab\\.local
          # Allow SSL termination at reverse proxy, run HTTP internally
          - extra_params=--o:ssl.enable=false --o:ssl.termination=true
          # Optional admin console
          - username=admin
          - password=CHANGE_ME_COLLABORA_ADMIN
        cap_add:
          - MKNOD
        # Logs/config can be persisted later if needed; CODE is mostly stateless.
        # Example:
        # volumes:
        #   - /srv/docker/nextcloud/collabora:/config
        ports:
          - "9980:9980"
        networks:
          - nextcloud-net

    networks:
      nextcloud-net:
        driver: bridge

**Important:** Replace all `CHANGE_ME_*` values with proper strong secrets before you run `docker compose up -d`.

---

## 7. Deployment Steps

From the Docker host:

    cd /srv/docker/nextcloud
    docker compose pull
    docker compose up -d

Check:

    docker ps | grep nextcloud
    docker ps | grep collabora

If everything’s healthy:

    docker logs nextcloud-app --tail=50
    docker logs collabora --tail=50

You should see Nextcloud finishing installation and Apache listening on port 80 (mapped to host 8081).

---

## 8. First Run — Nextcloud

Browse to:

    http://<docker-host-ip>:8081/

Because we set `NEXTCLOUD_ADMIN_USER` and `NEXTCLOUD_ADMIN_PASSWORD`, you should **skip** the DB/admin wizard entirely and land straight at the login page:

- Username: `ncadmin`
- Password: the value you set in `NEXTCLOUD_ADMIN_PASSWORD`

On first login:

1. Confirm language/timezone/user details.  
2. Optionally disable bundled “recommendation” apps you don’t want.  
3. Go to **Settings → Basic settings → Email server** and configure outbound mail (for password resets, sharing, etc.) when you’re ready.

At this point, files/sync are usable.

---

## 9. First Run — Collabora Integration (Nextcloud WebUI)

From the Nextcloud WebUI (as `ncadmin`):

1. Open your user menu → **Apps**.  
2. Find and install **“Collabora Online”** (and “Richdocuments” if shown separately).  
3. After install, go to:

       Settings → Administration → Office / Collabora Online

4. Set:
   - **Use your own server**  
   - Server URL:

       http://collabora:9980

   This works because `nextcloud-app` and `collabora` share the `nextcloud-net` Docker network and can reach each other by service name.

5. Save and test by opening a new document from the Files app.

Later, when the reverse proxy is in place, you can optionally switch this to:

- External URL: `https://office.lab.local`

…with Collabora also being accessed via NPM. For zero-touch right now, the internal `http://collabora:9980` endpoint is fine.

---

## 10. Reverse Proxy (when you’re ready)

Once Nginx Proxy Manager (or equivalent) is live:

- Create a proxy host for Nextcloud:
  - `cloud.lab.local` → `http://nextcloud-app:80` (or Docker host `8081`)
  - Enable SSL and HSTS as appropriate
- Create a proxy host for Collabora:
  - `office.lab.local` → `http://collabora:9980`
  - Ensure websockets and large headers are allowed

Then update:

- Internal DNS so `cloud.lab.local` and `office.lab.local` point at the reverse proxy.  
- In Nextcloud’s Office settings, change Collabora URL to `https://office.lab.local`.

The `NEXTCLOUD_OVERWRITE*` env settings already assume HTTPS at the proxy.

---

## 11. Backup & Restore

Backup at least:

    /srv/docker/nextcloud/app
    /srv/docker/nextcloud/db
    /srv/docker/nextcloud/redis   # optional, can be rebuilt
    /srv/docker/nextcloud/collabora   # only if you end up persisting config there

Common strategy:

- Stop the stack briefly (for consistent DB/files):

      cd /srv/docker/nextcloud
      docker compose down

- Snapshot / backup the directories above.  
- `docker compose up -d` afterwards.

Restore sketch:

1. Recreate Docker Host HSL.  
2. Lay down `/srv/docker/nextcloud` contents from backup.  
3. Recreate `docker-compose.yml` (or restore it).  
4. `docker compose up -d` from `/srv/docker/nextcloud`.  

Nextcloud should come back with files, users, shares, and Collabora integration intact.

---

## 12. Zero-Touch Checklist

After `docker compose up -d`, required “touch points” are:

- Nextcloud WebUI:
  - First admin login (using pre-set credentials)
  - Install “Collabora Online” app
  - Point it to `http://collabora:9980`  
- Optional:
  - Configure email in Nextcloud
  - Tighten app selection and theming

No manual edits of `config.php`, no DB shell, no container execs are required for the core setup.

Once this is in place, you’ve got:

- Files & sync: Nextcloud
- In-browser office: Collabora
- All sitting cleanly on the Docker Host HSL and ready for NPM/backup integration.

