# DNS — An overview
A practical, detailed, admin-friendly guide to DNS in a homelab.  
This document builds on *DNS — The Simple Version* and gives you solid mental models, correct terminology, and enough depth to diagnose and prevent common DNS issues.

---

## Purpose of This Guide
DNS is deceptively simple but foundational.  
When it’s correct, everything feels fast and predictable.  
When it’s wrong, every service suddenly appears “broken”, even though they aren’t.

This guide explains:
- what DNS actually does  
- how local and public DNS work together  
- how a DNS lookup flows from start to finish  
- what caching layers exist (and how they betray you)  
- how to troubleshoot DNS issues confidently in a homelab  

No theory for theory’s sake — just the knowledge that makes your environment stable and your debugging sane.

---

## 1. What DNS Actually Is
DNS is the distributed system that translates **human names** into **IP addresses**.

Example:
```
unifi.home → 10.10.1.21
github.com → 140.x.x.x
```

This lookup is what allows:
- browsers to find websites  
- SSH to reach your servers  
- applications to talk to each other  
- certificates to validate domains  
- your homelab dashboards to load  

Without DNS, everything falls back to raw IPs, which quickly becomes chaos.

---

## 2. Local vs Public DNS (Practical Mental Model)

Your homelab relies on two DNS “universes”:

### Local DNS
Handles:
- homelab servers  
- containers  
- internal apps  
- internal service discovery  

Common setups:
- OPNsense + Unbound  
- Pi-hole + Unbound  
- Active Directory DNS  

Local DNS knows your *private networks*.

### Public DNS
Handles:
- websites  
- software updates  
- external APIs  
- cloud services  

Common resolvers:
- Cloudflare: 1.1.1.1  
- Google: 8.8.8.8  
- Quad9: 9.9.9.9  

Public DNS knows the *internet*.

### When something breaks
- Local DNS down → your homelab disappears  
- Public DNS down → the internet disappears  
- Both down → enjoy an intense character-building afternoon  

---

## 3. How DNS Resolution Works (Step-by-Step)

When you query a name, the system works down these layers:

### 1. Local cache
The OS remembers recent lookups.  
This avoids asking DNS unnecessarily.

### 2. Hosts file
Manual overrides live here:
- Linux/macOS → /etc/hosts  
- Windows → C:\Windows\System32\drivers\etc\hosts  

If present, this **wins** over DNS.

### 3. Configured DNS servers
The OS asks the resolvers listed in:
/etc/resolv.conf         (Linux)
Network Adapter DNS      (Windows)

### 4. Recursive lookup
If your resolver doesn’t know the answer, it forwards your question upstream.

Unbound, Pi-hole and most home resolvers handle recursion automatically.

### 5. Return the final answer
Once resolved, the system stores the IP for the duration of its TTL.

This layered process is why DNS may appear “wrong” even when your server is correct — caches often tell yesterday’s story.

---

## 4. DNS Records You’ll Meet in a Homelab

### A Record
Maps a name → IPv4 address  
Example:
nas.home    A    10.10.1.50

### AAAA Record
Maps a name → IPv6 address  
(Not required unless you use IPv6.)

### CNAME
Alias pointing to another hostname:
ha.home     CNAME   homeassistant.local

### PTR
Reverse lookup:
10.10.1.50 → nas.home

Useful for logs and AD environments.

### SRV Records
Service discovery:
- used in AD  
- some apps rely on them  

Most homelabs only need A, AAAA, and CNAME.

---

## 5. DNS Servers in a Homelab (Roles & Behaviour)

A typical setup looks like:

### 1. Primary resolver (Unbound, AD DNS, etc.)
Handles internal name resolution and forwards external queries.

### 2. DHCP server
Provides DNS server details to clients.

### 3. Forwarders
Public resolvers your local DNS uses when it doesn’t know an answer.

Example:
Local DNS → 1.1.1.1 → Root → TLD → Authoritative server → answer

### 4. Optional: Pi-hole
Adds filtering and ad-blocking.

### 5. Optional: Split horizon DNS
Internal names resolve to internal IPs.  
External names resolve to public IPs.

---

## 6. Caching — The Source of Half Your Problems

DNS has *multiple* caches:
- Browser cache  
- OS resolver cache  
- Local DNS server cache  
- Upstream resolver cache  
- Application cache  

This means:
- a DNS change isn’t instant  
- devices may update at different speeds  
- containers may use completely different DNS resolvers  

Testing directly against your DNS server avoids cache confusion:
dig @10.10.1.1 unifi.home

---

## 7. How Containers Handle DNS (Important)

Docker and Podman often rewrite DNS settings:
- default resolver → 127.0.0.11  
- internal DNS rules differ from the host  
- containers may not see .local mDNS names  
- containers may bypass Pi-hole or Unbound entirely  

When a container resolves differently from the host:
- check /etc/resolv.conf inside the container  
- set DNS explicitly in Docker Compose  
- ensure container networks are configured correctly  

---

## 8. DNS Troubleshooting (Practical Model)

### Test 1: Does it work by IP?
If yes → DNS problem  
If no → network or service problem  

### Test 2: Query your DNS server directly
dig @10.10.1.1 nas.home

### Test 3: Check what your system thinks
getent hosts nas.home

### Test 4: Check hosts file
Accidental entries cause days of pain.

### Test 5: Check container DNS
Especially if the issue only happens in containers.

### Test 6: Check cache freshness
You can flush caches if needed:
sudo systemd-resolve --flush-caches
or restart your DNS resolver.

---

## 9. Common DNS Issues and How to Fix Them

### 1. Wrong IP in DNS
Fix:
- update record  
- flush caches  
- verify with:  
dig @dns-server name

### 2. DNS server unreachable
Fix:
- check the gateway  
- check firewall rules  
- confirm the DNS service is running  
- verify LAN connectivity  

### 3. Stale DNS cache
Fix:
- flush local cache  
- restart DNS resolver  
- reduce TTLs for rapidly-changing internal services  

### 4. Split-horizon mismatch
External and internal clients get different answers.

Fix:
- ensure internal DNS overrides external records  
- avoid duplicating zones across systems  

### 5. Docker resolving incorrectly
Fix:
- set correct dns: values in Compose  
- ensure containers use Pi-hole/Unbound  
- check Docker network behaviour  

### 6. Hosts file overrides
Fix:
- remove/comment stale entries  

---

## 10. Worked Examples

### Example 1: Internal service doesn’t load by name
dig @10.10.1.1 it-tools.home
getent hosts it-tools.home

If results differ → stale cache or wrong record.

### Example 2: Containers can’t resolve internal names
Steps:
1. Check /etc/resolv.conf in the container  
2. Specify DNS in Compose  
3. Restart the container  

### Example 3: A DNS change not taking effect
sudo systemd-resolve --flush-caches
dig @10.10.1.1 <record>

---

## 11. Summary

DNS is simple to understand but easy to misconfigure.

Key points:
- DNS maps names to IPs  
- Local + public DNS serve different roles  
- Caches cause most confusion  
- Containers often use different DNS  
- Always test with IP first  
- Use dig to see the real truth  
- Consistent DNS = a calmer homelab  

---

A separate Advanced DNS guide builds on this foundation and covers additional record types (A, AAAA, MX, TXT, etc.), Active Directory–integrated DNS, and the practical nuances of DNS behaviour that are not typically covered in introductory or standard reference material.

---

## 12. Appendix

### Useful Commands
dig <name>
dig @<dns-server> <name>
nslookup <name>
getent hosts <name>
sudo systemd-resolve --flush-caches

### Record Reference
| Record | Purpose |
|--------|---------|
| A     | IPv4 address |
| AAAA  | IPv6 address |
| CNAME | Alias |
| PTR   | Reverse lookup |
| SRV   | Service (mainly AD) |
