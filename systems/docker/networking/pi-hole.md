# Pi-hole — Service Pattern  
*A defined DNS sinkhole, not a mystery box of ad-blocking.*

---

## Pattern Overview

- **Pattern name:** Pi-hole  
- **Host Service Layer:** Docker on Linux as a Service Host  
- **Category:** DNS / Network Services  
- **Exposure:** Internal LAN only  
- **Criticality:** High (core DNS for clients that use it)  
- **Intended users:** Homelab operator / network administrator  

This pattern assumes the Docker Host HSL is already in place:

- `/srv/host-docker` on dedicated disk  
- `/srv/docker` and `/srv/logs` structured  
- Docker Root Dir relocated from default  

Pi-hole becomes a defined DNS service with its own IP, not an ad-hoc container running “somewhere”.

---

## 1. Purpose

Pi-hole provides:

- Network-wide DNS-level ad blocking  
- Centralised DNS query logging and statistics  
- Local override/host mapping  
- Optional per-client or per-network policy control  

This pattern ensures:

- Predictable deployment  
- Persistent configuration and blocklists  
- Clear separation of app data vs logs  
- A repeatable way to introduce DNS-level filtering into the homelab

---

## 2. When to Use / When Not to Use

### 2.1 When to use this pattern

Use Pi-hole when:

- You want DNS-based ad / tracker blocking for one or more VLANs  
- You want visibility of DNS queries from internal clients  
- You prefer a lightweight DNS sinkhole over browser extensions alone  
- You want a clearly documented, redeployable DNS appliance  

Typical fits:

- OPNsense LAN/VLANs pointing at Pi-hole as primary DNS  
- Single “house” network using Pi-hole as DNS, forwarding to upstream resolvers  
- Lab environments where you want to quickly test DNS policies

### 2.2 When not to use this pattern

This pattern is not ideal if:

- You require enterprise-grade DNS with HA and advanced policy (use BIND/Unbound/PowerDNS clusters etc.)  
- You are already using a different internal DNS strategy that you don’t want to change  
- You do not control DHCP / DNS handed out to clients (e.g. ISP router locked down)  

In those cases, Pi-hole may still be useful on a subset of devices, but not as a full-network default.

---

## 3. Prerequisites

We assume:

- A Docker Host following the Hvelvigen baseline  
- You control at least one of:
  - DHCP (e.g. via OPNsense)  
  - Manual DNS configuration on your key clients  
- You are comfortable with:
  - basic shell usage  
  - docker compose  
  - basic DNS concepts  

This pattern uses a **macvlan network** so that the Pi-hole container has its own IP address on the LAN.  
That allows you to configure DNS on clients/firewall as if Pi-hole were a standalone appliance.

You will need:

- The parent interface name on the host (for example: `enp3s0`)  
- The LAN subnet and gateway (for example: `10.16.1.0/24` and `10.16.1.1`)  
- A spare IP address on that subnet to dedicate to Pi-hole (for example: `10.16.1.21`)  

All values below are placeholders and should be replaced with your actual details.

---

## 4. Service Design

High-level design:

- A single container running `pihole/pihole`  
- Bind-mounted persistent storage for:
  - Pi-hole configuration and gravity DB  
  - dnsmasq configuration  
- A dedicated **macvlan** network that:
  - attaches the container directly to the LAN  
  - assigns it its own IP address  
  - lets clients and the firewall treat it as a normal DNS server  

Service role:

- Acts as a DNS recursive forwarder / sinkhole for configured clients  
- Forwards queries upstream to your chosen resolvers (e.g. Unbound, DoT/DoH resolver, public DNS)  
- Provides UI for blocklists, custom records, and stats  

Note: with macvlan, the **host cannot directly reach the container IP** by default. Access the Pi-hole IP from other LAN devices. If host-to-Pi-hole access is required, a host-side macvlan shim interface can be added later (intentionally out of scope for this pattern).

---

## 5. Environment & Configuration Inputs

Pi-hole uses environment variables to set:

- Time zone  
- Web UI password  
- Advertised IPv4 address  

Minimum recommended:

- `TZ` — time zone (e.g. `Europe/London`)  
- `WEBPASSWORD` — initial admin UI password  
- `FTLCONF_LOCAL_IPV4` — Pi-hole’s LAN IP address  

You can also set upstream DNS servers via environment or within the UI after first run.

---

## 6. Directory Preparation

Base path:

- `/srv/docker/pihole`

Create directories:

    sudo mkdir -p /srv/docker/pihole/etc-pihole
    sudo mkdir -p /srv/docker/pihole/etc-dnsmasq.d

Optional logs directory:

    sudo mkdir -p /srv/logs/pihole

Set ownership (UID/GID may be adjusted to your standard service user):

    sudo chown -R 1000:1000 /srv/docker/pihole

Result:

    /srv/docker/pihole
    ├── etc-pihole/     # Core Pi-hole data and configuration
    └── etc-dnsmasq.d/  # dnsmasq configuration snippets

---

## 7. Docker Compose (Dedicated LAN IP via macvlan)

File:

- `/srv/docker/pihole/docker-compose.yml`

Change the placeholders in angle brackets to match your environment:

- `<PARENT_IF>`    – host NIC on the LAN (for example `enp3s0`)  
- `<SUBNET_CIDR>`  – LAN subnet (for example `10.16.1.0/24`)  
- `<GATEWAY_IP>`   – default gateway on that subnet (for example `10.16.1.1`)  
- `<PIHOLE_IP>`    – static IP to assign to Pi-hole (for example `10.16.1.21`)  
- `<IP_RANGE>`     – optional smaller range for macvlan (for example `10.16.1.192/28`)  
- `<WEBPASSWORD>`  – initial Pi-hole admin password  

### 7.1 Create the file

    cd /srv/docker/pihole
    nano docker-compose.yml

Add:

    services:
      pihole:
        image: pihole/pihole:latest
        container_name: pihole
        restart: unless-stopped

        networks:
          pihole_net:
            ipv4_address: <PIHOLE_IP>

        environment:
          - TZ=Europe/London
          - WEBPASSWORD=<WEBPASSWORD>
          - FTLCONF_LOCAL_IPV4=<PIHOLE_IP>

        volumes:
          - /srv/docker/pihole/etc-pihole:/etc/pihole
          - /srv/docker/pihole/etc-dnsmasq.d:/etc/dnsmasq.d

        # DNS from the container's perspective (upstream resolvers):
        # environment:
        #   - DNS1=1.1.1.1
        #   - DNS2=1.0.0.1
        #
        # or point at an internal resolver if you prefer.

    networks:
      pihole_net:
        driver: macvlan
        driver_opts:
          parent: <PARENT_IF>
        ipam:
          config:
            - subnet: "<SUBNET_CIDR>"
              gateway: "<GATEWAY_IP>"
              ip_range: "<IP_RANGE>"

Notes:

- No `ports:` section — Pi-hole listens on port 53 (TCP/UDP) and 80 directly on `<PIHOLE_IP>`.  
- Make sure `<PIHOLE_IP>` is outside your DHCP pool and not in use by another device.  

Save and exit.

### 7.2 Bring the service up

    docker compose up -d

Verify:

    docker ps | grep pihole
    docker logs pihole --tail=50

If the network or IP is misconfigured, macvlan-related errors will show in the logs.

---

## 8. First Run & Initial Access

From another device on the LAN (not the Docker host), browse to:

    http://<PIHOLE_IP>/admin

Example:

    http://10.16.1.21/admin

Log in with:

- Password: `<WEBPASSWORD>` (as set in the compose file)  

Immediately after first login:

- Confirm **time zone** is correct  
- Configure preferred upstream DNS servers (if not set via environment)  
- Review and adjust blocklists as needed  

---

## 9. Logging

Current behaviour:

- Container logs via:

      docker logs pihole

- Query logs and long-term statistics are stored within:

      /srv/docker/pihole/etc-pihole

Optional later:

- Bind-mount specific log files into `/srv/logs/pihole` for:
  - centralised log rotation  
  - log shipping to external tools  

You can adjust Pi-hole’s logging level from the web UI if you want to reduce noise.

---

## 10. Updating

To update Pi-hole:

    cd /srv/docker/pihole
    docker compose pull
    docker compose up -d

Then:

- Confirm the UI is reachable at `http://<PIHOLE_IP>/admin`  
- Check that DNS queries are being answered correctly from a client pointing at `<PIHOLE_IP>`  
- Watch `docker logs pihole` for any start-up or FTL errors  

---

## 11. Backup

To restore Pi-hole cleanly you need to back up at least:

    /srv/docker/pihole/etc-pihole
    /srv/docker/pihole/etc-dnsmasq.d

These contain:

- Query logs and statistics  
- Blocklists and gravity database  
- Custom DNS records and overrides  
- dnsmasq configuration snippets  

Recommended:

- Include Pi-hole in your infrastructure backup set  
- Take a backup before major list/policy changes  
- Periodically validate that a restore onto another host works as expected  

Restore process (high level):

1. Restore the directories to `/srv/docker/pihole`  
2. Recreate the stack with `docker compose up -d`  
3. Confirm that DNS and the web UI behave as expected  

---

## 12. Integration

Common integration patterns:

- **OPNsense / DHCP integration:**
  - Set `<PIHOLE_IP>` as the primary DNS server in OPNsense DHCP for selected VLANs.  
  - Optionally keep a secondary DNS (e.g. OPNsense itself) if you want a fallback.

- **Static configuration on critical clients:**
  - Point key devices (admin workstation, test machines) directly at `<PIHOLE_IP>` as their DNS server.  

- **DNS forwarding chain:**
  - Pi-hole forwards to chosen upstreams (public DNS, DoT/DoH forwarder, or an internal resolver).  

Security / exposure:

- Pi-hole should be **LAN-only**. Do not expose port 53 or the web UI to the public internet.  
- Consider restricting management of the web UI to a management subnet or VPN if desired.  

---

With this pattern, Pi-hole:

- Lives at a predictable IP on the LAN  
- Is fully reproducible with a documented compose definition  
- Stores all important state under `/srv/docker/pihole`  
- Can be cleanly backed up, rebuilt, or migrated without guesswork.
