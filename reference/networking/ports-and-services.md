# Ports & Services — Full Version (Practical Admin Depth)

A practical, detailed, admin-friendly guide to understanding ports, listeners, and service access in a homelab.  
This document builds on the Simple Version and gives you the mental models and vocabulary you need to diagnose access issues confidently.

---

## Purpose of This Guide
Ports and services underpin everything in a homelab — dashboards, SSH, Docker containers, APIs, reverse proxies, you name it.  
Yet they’re often misunderstood, especially when things work locally but not remotely.

This guide aims to give you:
- a clear mental model for how ports work  
- an understanding of listening addresses and bindings  
- practical knowledge of how Docker port mapping behaves  
- the ability to diagnose access failures in minutes  
- confidence to run multiple services on the same machine  

No heavy protocol theory — just what you actually need.

---

## 1. What a Port Actually Is
Every device uses ports to run multiple networked services simultaneously.

Think of the device as the building (IP address), and ports as the numbered doors leading to specific rooms (services).

Examples:
```
22   SSH
80   HTTP
443  HTTPS
3000 Admin tools
8080 Dashboards and web UIs
```

Each port represents:
- a number (0–65535)  
- a transport protocol (TCP or UDP)  
- a process listening for connections  

If no process is listening, the port is closed — nothing can reach the service.

---

## 2. The Relationship Between IP, Port, and Service
A complete destination is:
```
<IP address>:<port>
```

This combination is called a **socket**.

Examples:
```
10.10.1.20:22     → SSH
10.10.1.50:3000   → Admin UI
192.168.1.10:443  → HTTPS
```

Services listen on sockets, not just ports.  
This distinction becomes important later when we talk about binding and listening addresses.

---

## 3. Listening Addresses (Binding Behaviour)
A service can choose **where** it listens.  
This explains almost every “works locally but not over the network” moment.

### 127.0.0.1:<port>
Service listens only on loopback.  
Accessible only from the same machine.

Useful for:
- local-only tools  
- reverse proxies  
- security-by-isolation setups  

---

### <LAN IP>:<port>
Service listens only on that interface.

Example:
```
192.168.1.10:8080
```

Useful for:
- multi-homed servers  
- limiting exposure to a specific network  

---

### 0.0.0.0:<port>
Service listens on **all** IPv4 interfaces.

Useful for:
- web servers  
- dashboards  
- anything that should be reachable across your LAN  

This is the default for many tools.

---

### Common Symptom
- Works on the device  
- Doesn’t work from anywhere else  

**Cause:** bound to 127.0.0.1 instead of LAN IP.

---

## 4. How Ports and Firewalls Interact
Even if a service listens correctly, the firewall can block it.

### UFW example
```
sudo ufw allow 8080/tcp
```

### OPNsense / other firewalls
May block:
- inbound WAN ↔ LAN  
- inter-VLAN traffic  
- east-west traffic inside LANs depending on rules  

Firewalls don’t care that the service is listening.  
They only care about whether traffic is allowed through.

---

## 5. Docker Port Mapping (Practical Mental Model)
Docker adds a layer of indirection:

```
-p 8080:80
```

Means:
- **host** listens on port 8080  
- **container** listens on port 80  
- accessing http://host-ip:8080 reaches container port 80  

You never connect directly to the container — always to the **host**, which forwards traffic.

This becomes crucial when:
- multiple containers need port 80 or 443  
- you run a reverse proxy  
- you run stacks on the same server  

Docker networking is powerful but predictable once you internalise what is exposed *where*.

---

## 6. Checking What’s Listening (Your Most Useful Tool)
Linux provides an excellent socket inspection tool:

```
ss -tuln
ss -tulnp   # includes process names
```

This tells you:
- which ports are open  
- whether they’re TCP/UDP  
- which addresses they are bound to  
- which process owns them  

If a port isn’t listed, **nothing is listening**, even if you’re certain the service “should be running”.

---

## 7. Multiple Services and Port Conflicts
Two services cannot listen on the same port on the same IP.

Symptoms:
- one service refuses to start  
- errors like “address already in use”  
- service randomly unreachable

Common causes:
- two containers both exposing 80  
- a reverse proxy already using 443  
- leftover services from testing  

Fix:  
Change the port or use Docker mapping to remap to unused ports.

---

## 8. How Reverse Proxies Fit In
Reverse proxies (NGINX, Traefik, Caddy) typically bind to:
```
80  (HTTP)
443 (HTTPS)
```

They handle:
- TLS termination  
- virtual hosts  
- routing to backend apps  
- clean URLs  

Your backend services often move to:
```
127.0.0.1:3000
127.0.0.1:8080
127.0.0.1:9000
```

This keeps them internal and secure.

---

## 9. Diagnostics and Troubleshooting

### 1. Service works locally but not remotely
Likely cause:
- bound to 127.0.0.1  
- firewall blocking external access  

Check with:
```
ss -tulnp
```

---

### 2. Browser can’t reach dashboard
Check the actual port the app listens on.  
Some services default to unusual ports (e.g., 8123 for Home Assistant).

---

### 3. Docker container unreachable
Check:
- port mapping  
- container logs  
- whether container binds to 0.0.0.0  
- Docker network mode  

---

### 4. Two services fighting for the same port
Look for:
```
address already in use
```
Fix:
- stop the other service  
- change the port  
- move behind reverse proxy  

---

### 5. Firewall blocking
Test:
```
curl -v http://server-ip:port
```
If it works locally but not from another device → firewall.

---

## 10. Worked Examples

### Example A: A local dashboard
Service listens on:
```
0.0.0.0:8080
```
Access from:
```
http://10.10.1.20:8080
```

---

### Example B: A local-only service behind reverse proxy
Service:
```
127.0.0.1:3000
```
Reverse proxy:
```
:80
:443
```
Browser uses:
```
https://myservice.home
```

---

### Example C: Docker stack
Container listens on 80.  
You publish:
```
-p 8080:80
```
Access from LAN:
```
http://server-ip:8080
```

---

## 11. Summary
Ports and services follow simple rules:

- Port = doorway into a service  
- IP + Port = socket  
- Services choose where they listen  
- Docker can remap ports  
- Firewalls can block access  
- Binding decides who can connect  
- Conflicts prevent services from starting  

Once you understand these, service access issues become predictable and easy to resolve.

---

## 12. Appendix

### Useful Commands
```
ss -tuln
ss -tulnp
curl -v http://host:port
```

### Common Ports
| Port | Purpose |
|------|---------|
| 22   | SSH |
| 80   | HTTP |
| 443  | HTTPS |
| 3000 | Admin tools |
| 8080 | Dashboards / Web UIs |
| 8123 | Home Assistant |
