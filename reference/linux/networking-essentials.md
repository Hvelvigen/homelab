# Networking Essentials
A friendly, practical introduction to the core networking ideas you *actually* need when running a homelab.  
Just enough to understand what’s happening before you start poking firewalls, Docker ports, VLANs, or wondering why a service works on the server but not your laptop.

This guide stays intentionally simple — clean concepts, clear mental models, and a little humour so future‑you doesn’t scream into a cabinet.

---

## 1. What This Guide Covers
By the end, you’ll understand:

- what an IP address represents  
- how devices talk on a LAN  
- what “localhost” really is  
- why ports exist  
- how DNS converts names into IPs  
- how to troubleshoot basic “why isn’t this working‽” moments

Nothing here requires deep networking knowledge.  
This is the foundation for everything else.

---

## 2. Your Local Network (LAN)
Your LAN is your private network — your own little kingdom of packets.

Example:

```
10.10.1.1    # router / firewall (gateway)
10.10.1.20   # Docker/Ubuntu host
10.10.1.30   # NAS
10.10.1.50   # your PC
```

Devices in the same subnet (e.g., `10.10.1.x/24`) can talk directly unless a firewall stops them.

That’s the rule. No secret handshake required.

---

## 3. IP Addresses — What They Actually Represent
An IP address is a network address. Not magical. Not scary.

Example:

```
10.10.1.20
```

- `10.10.1` → the network (“the street”)  
- `20` → the device (“the house”)  

Add `/24` and it simply means:  
“Everything from 10.10.1.1 to 10.10.1.254 lives on the same street.”

---

## 4. The Gateway — How You Reach Anything Else
The gateway is the device that knows how to reach:

- the internet  
- other VLANs  
- VPNs  
- different subnets  

If it dies or is misconfigured, the world outside your LAN evaporates.

Quick test:

```
ping <gateway-ip>
```

If it doesn’t reply… well, that explains a lot.

---

## 5. DNS — Turning Names into IP Addresses
DNS is the internet’s phonebook.

```
unifi.home → 10.10.1.21
google.com → 142.x.x.x
```

If DNS lies, goes down, or returns stale results, everything *looks* broken even when it isn’t.

Docker pulls, browsers, updates — all rely on DNS behaving.

---

## 6. Localhost — What It Really Means
`localhost` = **this machine only**.  
Always.

```
127.0.0.1
```

So:

- “localhost on the server” ≠ “localhost on your PC”
- They are completely different universes

This explains 90% of the “It works here but not there!” moments.

---

## 7. Ports — Doorways to Services
IP = the device.  
Port = the specific service.

Examples:

```
22   SSH
80   HTTP
443  HTTPS
8080 Web dashboards
3000 Admin consoles
```

To reach a service:

```
http://10.10.1.20:8080
```

Docker uses port mapping to forward one doorway into another.

---

## 8. Listening — The Most Overlooked Concept
If nothing is listening on a port, nothing can reach it.

Check:

```
ss -tuln
```

Common bindings:

- `127.0.0.1:8080` → local-only  
- `10.10.1.20:8080` → LAN-only  
- `0.0.0.0:8080` → all interfaces  

These decide who *can* connect.

---

## 9. Troubleshooting Flow (Use This Every Time)
When something refuses to work:

1. **IP** — Does the device respond?  
2. **DNS** — Does the name resolve?  
3. **Port** — Is the service actually listening?  
4. **Listening address** — Who is allowed to reach it?  
5. **Firewall** — Are packets being blocked?

You will fix 90% of issues with these five steps.

---

## 10. Summary
The essentials:

- LAN = local devices talking directly  
- IP = device address  
- Gateway = your exit to anywhere else  
- DNS = names → IPs  
- Localhost = this device, never another  
- Ports = doorways to services  
- Listening = whether a service can be reached  
- Troubleshooting = **IP → DNS → Port → Listening → Firewall**

You now have the foundation.  
Everything deeper fits on top of this without tears.
