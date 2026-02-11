# Nginx Proxy Manager — Service Pattern  
*A reverse proxy with a proper home, not a pile of random port forwards.*

---

## Pattern Overview

- **Pattern name:** Nginx Proxy Manager  
- **Host Service Layer:** Docker on Linux as a Service Host  
- **Category:** Edge / Reverse Proxy  
- **Exposure:** Internal LAN (optionally WAN via firewall rule)  
- **Criticality:** High (authoritative ingress for HTTP/S)  
- **Intended users:** Homelab operator / network administrator  

This pattern assumes the Docker Host HSL is already in place:

- `/srv/host-docker` on dedicated disk  
- `/srv/docker` and `/srv/logs` structured  
- Docker Root Dir relocated from default  

NPM becomes the **front door of the host**.  
No other service binds directly to ports 80 or 443 on this machine.

---

## 1. Purpose

Nginx Proxy Manager provides:

- Centralised reverse proxy management  
- Clean FQDN-based routing for internal services  
- Automatic Let’s Encrypt certificate management  
- Per-host SSL configuration and redirects  
- A GUI for managing proxies, certificates, and access lists  

This pattern ensures:

- One canonical ingress point  
- No scattered port exposure  
- Predictable TLS termination  
- Clear separation between “edge” and “back-end” services  

All HTTP/S traffic terminates at NPM first.

---

## 2. When to Use / When Not to Use

### 2.1 When to use this pattern

Use Nginx Proxy Manager when:

- You want a single, authoritative ingress on the host IP  
- Multiple services need HTTPS exposure  
- You want all public access routed through one control point  
- You are running a single Docker host behind a firewall  

Typical fit:

- OPNsense WAN → forward 80/443 → Docker host → NPM → internal services

### 2.2 When not to use this pattern

This pattern is not ideal if:

- You require HA ingress clustering  
- You need automatic container label discovery  
- You are moving to Kubernetes-native ingress  
- You require fully declarative proxy configuration  

In those cases, look at Traefik, Caddy, or Kubernetes Ingress Controllers.

---

## 3. Prerequisites

We assume:

- Docker Host following the Hvelvigen baseline  
- Ports 80 and 443 are free on the host  
- Any existing services previously using 80/443 have been moved to alternate ports  
- You control DNS and firewall configuration  

Ports required on the host IP:

- 80/tcp – HTTP  
- 443/tcp – HTTPS  
- 81/tcp – NPM admin UI (LAN-only)

---

## 4. Service Design

High-level design:

- Single container running `jc21/nginx-proxy-manager`  
- Bind-mounted persistent storage  
- Host port bindings for 80/443/81  

Service role:

- Authoritative ingress for all HTTP/S traffic  
- TLS termination point  
- Reverse proxy to back-end services (Docker or non-Docker)  
- Centralised certificate lifecycle  

Back-end services:

- Do not bind to host 80/443  
- Typically expose high ports (for example 8080, 3000, 9000)  
- Or remain internal to Docker bridge networks  

---

## 5. Environment & Configuration Inputs

Configured primarily through the UI.

Recommended:

| Input                         | Where set           | Purpose                          |
|------------------------------|---------------------|----------------------------------|
| Admin email                  | First-login wizard  | Account + ACME                   |
| Admin password               | First-login wizard  | UI access                        |
| HTTP → HTTPS redirect policy | Settings            | Enforce TLS where appropriate    |
| Time zone                    | Host or env         | Log and certificate correctness  |

Optional at compose level:

- `TZ=Europe/London`

---

## 6. Directory Preparation

Base path:

- `/srv/docker/npm`

Create:

    sudo mkdir -p /srv/docker/npm/data
    sudo mkdir -p /srv/docker/npm/letsencrypt

Optional logs directory:

    sudo mkdir -p /srv/logs/npm

Set ownership:

    sudo chown -R 1000:1000 /srv/docker/npm

Result:

    /srv/docker/npm
    ├── data/
    └── letsencrypt/

---

## 7. Docker Compose (Host Front Door Model)

File:

- `/srv/docker/npm/docker-compose.yml`

### 7.1 Create the file

    cd /srv/docker/npm
    nano docker-compose.yml

Add:

    services:
      npm:
        image: jc21/nginx-proxy-manager:latest
        container_name: npm
        restart: unless-stopped

        ports:
          - "80:80"
          - "443:443"
          - "81:81"

        volumes:
          - /srv/docker/npm/data:/data
          - /srv/docker/npm/letsencrypt:/etc/letsencrypt

        # Optional:
        # environment:
        #   - TZ=Europe/London

Notes:
 
- NPM exclusively owns host ports 80 and 443.  
- Other services must not bind those ports.

Save and exit.

### 7.2 Bring the service up

    docker compose up -d

Verify:

    docker ps | grep npm
    docker logs npm --tail=50

---

## 8. First Run & Initial Access

Access via:

    http://<host-ip>:81

Example:

    http://10.22.1.56:81

Default credentials:

- Email: admin@example.com  
- Password: changeme  

Immediately:

1. Change credentials.  
2. Configure default HTTPS policy (for example redirect HTTP → HTTPS globally where appropriate).  
3. Confirm certificate issuance works for at least one test host.

---

## 9. Logging

Current:

    docker logs npm

Persistent data (including logs and SQLite DB) lives in:

    /srv/docker/npm/data

Optional later:

- Bind-mount logs into `/srv/logs/npm` if you want:
  - centralised log rotation  
  - log shipping to Wazuh, Zabbix, or similar  
- Add monitoring checks (for example Uptime Kuma) against:
  - `http://<host-ip>:81` for admin UI  
  - Representative proxied hosts  

---

## 10. Updating

Standard pattern:

    cd /srv/docker/npm
    docker compose pull
    docker compose up -d

Then verify:

- Admin UI reachable at `http://<host-ip>:81`  
- Proxy hosts operational via their FQDNs  
- No ACME or TLS errors in `docker logs npm`  

---

## 11. Backup

Backup:

    /srv/docker/npm/data
    /srv/docker/npm/letsencrypt

These include:

- Proxy host definitions  
- Users and access lists  
- Certificates and ACME state  

Restore (high level):

1. Restore directories to `/srv/docker/npm`.  
2. Run `docker compose up -d` in `/srv/docker/npm`.  
3. Validate routes and certificates behave as expected.

Include NPM in your normal infrastructure backup schedule.

---

## 12. Integration

Internal DNS:

- Point service hostnames (for example `nextcloud.lab.local`, `jellyfin.lab.local`) to the Docker host IP.  
- NPM routes traffic to the correct back-end containers.

Edge firewall (for example OPNsense):

- WAN 80/443 → Docker host IP 80/443.  
- Do not forward 81 from WAN. Keep admin UI LAN/VPN only.

Security:

- No public exposure of port 81.  
- All external HTTP/S terminates at NPM.  
- Back-end services remain unexposed directly to the internet.  
- Use access lists for admin or sensitive front-ends where appropriate.

---

With this pattern, Nginx Proxy Manager:

- Owns the host’s HTTP/S edge.  
- Eliminates scattered port exposure.  
- Centralises TLS and certificate management.  
- Simplifies firewall rules and DNS routing.  
- Provides a clean, authoritative ingress model for a single Docker host.
