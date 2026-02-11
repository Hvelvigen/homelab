# Jellyfin — Service Pattern  
*Media server with a clean home on the data disk.*

---

## Pattern Overview

**Pattern name:** Jellyfin  
**Host Service Layer:** Docker on Linux as a Service Host  
**Category:** Media / Streaming  
**Exposure:** Internal LAN (optionally via reverse proxy)  
**Criticality:** Low–Medium (nice to have, not infra-critical)  
**Intended users:** Homelab operator + household users  

Assumes the **Docker Host HSL** is already in place:

- Docker root moved to the data disk (`/srv/host-docker`)
- `/srv/docker` and `/srv/logs` structured
- Proxmox / OPNsense / core network up and stable

Jellyfin will:

- Store its own config/cache under `/srv/docker/jellyfin`
- Read media from `/mnt/media` (host) mounted into `/media` (container)
- Use `music/`, `movies/`, and `shows/` subfolders on the data disk (`sdb`)

---

## 1. Mount `/mnt/media` on the right disk (sdb)

There are two common scenarios.

### 1.1 If `sdb` is a dedicated media disk (new / empty)

1. Identify the disk:

    ```bash
    lsblk -f
    ```

   Confirm which device is your media disk (assume `/dev/sdb` here).

2. Partition it (if needed):

    ```bash
    sudo parted /dev/sdb -- mklabel gpt
    sudo parted -a opt /dev/sdb -- mkpart primary ext4 0% 100%
    ```

3. Create filesystem:

    ```bash
    sudo mkfs.ext4 -L MEDIA /dev/sdb1
    ```

4. Create mountpoint:

    ```bash
    sudo mkdir -p /mnt/media
    ```

5. Get the UUID:

    ```bash
    sudo blkid /dev/sdb1
    ```

   Note the `UUID="..."` value.

6. Add to `/etc/fstab`:

    ```bash
    sudo nano /etc/fstab
    ```

   Add a line (replace `YOUR-UUID-HERE` with the actual UUID):

    ```bash
    UUID=YOUR-UUID-HERE  /mnt/media  ext4  defaults,noatime  0  2
    ```

7. Mount it:

    ```bash
    sudo mount -a
    ```

8. Create the folder structure:

    ```bash
    sudo mkdir -p /mnt/media/movies
    sudo mkdir -p /mnt/media/shows
    sudo mkdir -p /mnt/media/music
    ```

---

### 1.2 If `sdb` already backs `/srv` (Docker/data disk)

In that case, you can keep the data physically on `sdb` under `/srv`, and bind-mount it to `/mnt/media` for a clean path.

1. Create the media directory on the data disk:

    ```bash
    sudo mkdir -p /srv/media/movies
    sudo mkdir -p /srv/media/shows
    sudo mkdir -p /srv/media/music
    ```

1. Create `/mnt/media`:

    ```bash
    sudo mkdir -p /mnt/media
    ```

2. Add a bind-mount to `/etc/fstab`:

    ```bash
    sudo nano /etc/fstab
    ```

   Add:

    ```bash
    /srv/media  /mnt/media  none  bind  0  0
    ```

3. Mount:
4. 
    ```bash
    sudo mount -a
    ```

Now `/mnt/media` lives on `sdb` via `/srv/media`, but Jellyfin only ever sees `/mnt/media`.

---

## 2. Service Directories

On the Docker host:

- Service home: `/srv/docker/jellyfin`
- Logs (reserved): `/srv/logs/jellyfin`

Create:

    sudo mkdir -p /srv/docker/jellyfin/config
    sudo mkdir -p /srv/docker/jellyfin/cache
    sudo mkdir -p /srv/logs/jellyfin

If you map Jellyfin to run as UID/GID 1000 (typical service user):

    sudo chown -R 1000:1000 /srv/docker/jellyfin
    sudo chown -R 1000:1000 /srv/logs/jellyfin
    sudo chown -R 1000:1000 /mnt/media

(Adjust UID/GID if you use a different service account.)

---

## 3. Docker Compose Definition

**File:** `/srv/docker/jellyfin/docker-compose.yml`

Create the file:

    cd /srv/docker/jellyfin
    nano docker-compose.yml

Add:

    services:
      jellyfin:
        image: jellyfin/jellyfin:latest
        container_name: jellyfin
        restart: unless-stopped
        
        environment:
          - TZ=Europe/London
          # Optional but handy for reverse proxy:
          # - JELLYFIN_PublishedServerUrl=http://jellyfin.lab.local

        volumes:
          # Jellyfin configuration and library database
          - /srv/docker/jellyfin/config:/config
          # Transcoding / cache (kept on fast storage if possible)
          - /srv/docker/jellyfin/cache:/cache
          # Media library (read-only is usually fine)
          - /mnt/media:/media:ro

        ports:
          # HTTP
          - "8096:8096"
          # Optional HTTPS direct from Jellyfin (usually not needed if using NPM)
          # - "8920:8920"
          # Optional local discovery (only if you want UPnP/auto-discovery)
          # - "7359:7359/udp"
          # - "1900:1900/udp"

        networks:
          - jellyfin-net

    networks:
      jellyfin-net:
        driver: bridge

Notes:

- `/media` inside the container will have:
  - `/media/movies`
  - `/media/shows`
  - `/media/music`
- You can add more folders later; Jellyfin will pick them up when you add libraries.

---

## 4. Deploy Jellyfin

From the Docker host:

    cd /srv/docker/jellyfin
    docker compose pull
    docker compose up -d

Check it’s running:

    docker ps | grep jellyfin
    docker logs jellyfin --tail=50

---

## 5. First-Run Configuration (WebUI)

Browse to:

    http://<docker-host-ip>:8096

Initial wizard:

1. Choose language and admin account credentials.  
2. Add libraries:

   - Movies:
     - Content type: Movies
     - Folder: `/media/movies`
   - TV Shows:
     - Content type: TV Shows
     - Folder: `/media/shows`
   - Music:
     - Content type: Music
     - Folder: `/media/music`

3. Confirm metadata language, metadata providers (or keep defaults).  
4. Finish setup; Jellyfin will start scanning the library.

You can later:

- Move access to a reverse proxy (e.g. `https://jellyfin.lab.local`)
- Add hardware acceleration (VAAPI/QuickSync) by passing through `/dev/dri` and adjusting permissions if the host supports it.

---

## 6. Operations

**Logs (current):**

    docker logs -f jellyfin

`/srv/logs/jellyfin` is reserved for future file-based logging if you decide to centralise logs.

**Update:**

    cd /srv/docker/jellyfin
    docker compose pull
    docker compose up -d

**Backup:**

Include at least:

    /srv/docker/jellyfin/config

in your backups; media itself lives in `/mnt/media`.

---

With this, Jellyfin is:

- Fully hosted off the data disk (`sdb`)
- Cleanly separated (`/srv/docker/jellyfin` for app, `/mnt/media` for content)
- Ready to plug into your existing reverse proxy and monitoring stack.
