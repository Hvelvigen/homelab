# DNS — Foundations
A clear, beginner-friendly explanation of what DNS (Domain Name System) is and why everything falls apart the moment it sneezes.

---

## 1. What DNS Actually Does
DNS turns names into IP addresses.  
That’s it. That’s the whole job.

```
unifi.home → 10.10.1.21
google.com → 142.x.x.x
```

Computers love numbers.  
Humans do not.  
DNS keeps everyone speaking politely.

If DNS vanished, you’d be typing IPs like it’s dial-up era cosplay.

---

## 2. Local DNS vs Public DNS
Your homelab uses two kinds of DNS:

- **Local DNS** → resolves your internal stuff  
- **Public DNS** → resolves the outside world  

When local DNS breaks → your services disappear.  
When public DNS breaks → the internet disappears.  
When both break → welcome to “why isn’t anything working” season.

---

## 3. The Tools You’ll Actually Use

```
nslookup <name>
dig <name>
getent hosts <name>
```

`getent` is especially useful because it shows how *your system* interprets a name — handy when caches are telling creative stories.

---

## 4. Common DNS Mistakes

- The record points to an old IP  
- The DNS server is down or unreachable  
- A cache hasn’t expired yet  
- Typo in hostname  
- Docker containers quietly using a different DNS resolver  

If something works by IP but not by name → DNS is 99% the culprit.

---

## 5. The Hosts File — The Manual Override

Every system has a tiny, ancient DNS override:

```
/etc/hosts
C:\Windows\System32\drivers\etc\hosts
```

Example entry:

```
10.10.1.20 it-tools.local
```

Great for testing.  
Disastrous if forgotten later.

---

## 6. Summary

DNS is simple:

- Names → IPs  
- If the name fails, DNS is lying or broken  
- Always test with the IP  
- Caches will betray you  
- Docker may join in for fun  

Understand DNS and your homelab instantly becomes calmer.
