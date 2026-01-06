# IP Addressing — The Simple Version
A clear, beginner-friendly explanation of what IP addresses are and how they behave in a homelab.

---

## 1. What an IP Address Represents
An IP address is just a location on a network.

Think of it like:

- **street name** = the network  
- **house number** = the device  

Example:
```
10.10.1.42
```
- `10.10.1` → network  
- `42` → device  

That’s enough to get through your first 90 days of homelabbing without tears.

---

## 2. CIDR: The “Who Are My Neighbours?” Suffix
CIDR (`/24`) tells devices which addresses count as “local”.

```
10.10.1.42/24
```

Meaning:  
“Anyone from 10.10.1.1 to 10.10.1.254 is on my network.”

Anything outside that range needs routing via the gateway.

---

## 3. The Three Private IP Ranges
Your homelab will use one of:

```
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
```

These are reserved for internal use — no global collisions, no drama.

If you like tidy VLANs, `10.x.x.x` is the usual favourite.

---

## 4. Static vs DHCP vs Reservations
- **Static IP** → manually set; predictable  
- **DHCP** → automatic; unpredictable unless reserved  
- **DHCP reservation** → predictable *and* automated — the sweet spot  

Use reservations for almost everything.  
Keep static addressing for infrastructure or cases where the system must come up before DHCP is available.

---

## 5. Common IP Mistakes
- Two devices sharing an address  
- Incorrect subnet mask  
- Incorrect gateway  
- Expecting `localhost` to refer to another device  
- Waiting for DNS when it hasn’t updated yet  

These account for most beginner outages.

---

## 6. Quick Checks
```
ip a               # see your addresses
ping 10.10.1.1     # check gateway (assumes the gateway is 10.10.1.1)
ip route           # view routing table
```

If any of these look wrong, start there.

---

## 7. Summary
IP addressing is simple:

- Every device gets a unique address  
- CIDR decides who is local  
- The gateway handles everything else  
- Reservations keep life predictable  

You now understand the part most people manage to overthink.
