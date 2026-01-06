# LAN & Routing — Full Version (Practical Admin Depth)

A practical, admin-focused guide to how devices communicate inside and outside your homelab — with enough detail to make sense of routing decisions, but not enough to make you regret reading it.

---

## Purpose of This Guide

LAN behaviour and routing decisions sit underneath everything in your homelab:

- Docker containers  
- Hypervisors  
- Web dashboards  
- DNS  
- Storage  
- VLANs  
- VPNs  

When these fundamentals click, diagnosing “can’t reach X” becomes a calm, predictable exercise rather than a multi-hour emotional journey.

This guide gives you:

- a clear mental model for LAN communication  
- how routing decisions are actually made  
- how to read and trust the routing table  
- a practical understanding of VLAN segmentation  
- troubleshooting patterns that work every time  

No academia, no packet-capturing dissertations — just what helps in the real world.

---

## 1. How Devices Communicate on a LAN

When two devices are in the **same subnet**, they talk directly.  
No routing, no firewall hops, just neighbourly communication.

Example:
```
10.10.1.20  ↔  10.10.1.50
```

### Why this works

Devices perform a simple check:

> “Is this IP in my subnet?”

If the answer is **yes**, the device uses **ARP** to discover the MAC address of the target and fires frames directly across the LAN.

### ARP in practical English

ARP is effectively the LAN’s group chat:

```
Who has 10.10.1.50?
Reply to 10.10.1.20 please.
```

The owner responds with:

```
It’s me. Here’s my MAC address.
```

If ARP breaks, LAN communication breaks.  
Sometimes dramatically.  
Sometimes quietly and spitefully.

### LAN failures you’ll actually see

- Wrong subnet mask → devices disagree on who’s “local”  
- ARP cache stale → unreachable until refreshed  
- Duplicate IP → ARP flapping (your LAN has trust issues)  
- Wi-Fi isolation → clients politely pretending the LAN doesn’t exist  

---

## 2. When Routing Happens

If the target IP is **not** in the device’s subnet, it does *not* attempt ARP.  
Instead, it forwards traffic to the **gateway**.

The logic:

> “Destination isn’t local → I’ll let the gateway deal with it.”

This is how your device:

- reaches other VLANs  
- reaches the internet  
- reaches remote services  
- reaches misconfigured containers you swear you published correctly  

### Gateway requirements

Your gateway must:

- have an IP inside the subnet  
- answer ARP  
- be reachable  
- actually be a router (not a switch having an identity crisis)  
- know where to send onward traffic  

If any part of that fails, routing collapses faster than a Jenga tower.

---

## 3. The Routing Table — The Device’s Decision Maker

See it with:

```
ip route
```

Example:

```
default via 10.10.1.1 dev eth0
10.10.1.0/24 dev eth0 proto kernel scope link src 10.10.1.20
```

### How to read this

- `10.10.1.0/24` → “anything in my LAN goes directly out eth0”  
- `default via 10.10.1.1` → “everything else goes to the gateway”  

### Most specific route wins

If multiple routes could match an address, the one with the **longest prefix** (most matching bits) is chosen.

Example:

```
10.0.0.0/8
10.10.0.0/16
10.10.1.0/24  ← most specific
```

Devices are obedient.  
They do exactly what the routing table says — even if what the routing table says is… questionable.

---

## 4. VLANs — Segmented Networks, Same Cabling

VLANs let you split a single physical network into clean, isolated segments.

Typical homelab layout:

- Infrastructure  
- Servers  
- Clients  
- IoT  
- Guest  

Each VLAN gets:

- its own subnet  
- its own gateway  
- its own firewall rules  

### Golden rule

- **Same VLAN → direct communication**  
- **Different VLANs → routed communication**

If you break this rule, networking becomes interpretive dance.

### Why one subnet per VLAN matters

Mixing subnets inside a VLAN produces:

- ARP confusion  
- random reachability failures  
- intermittent DNS failures  
- packet loss so inconsistent you’ll question your life choices  

---

## 5. Inter-VLAN Routing

To move traffic between VLANs, you need a routing-capable device:

- L3 switch  
- Router  
- Firewall (most common in homelabs)  

Example mapping:

| VLAN | Subnet          | Gateway       |
|------|-----------------|---------------|
| 10   | 10.10.10.0/24   | 10.10.10.1    |
| 20   | 10.10.20.0/24   | 10.10.20.1    |
| 30   | 10.10.30.0/24   | 10.10.30.1    |

Firewall rules then decide who is allowed to talk to whom — and who must remain socially distanced.

---

## 6. Putting It All Together — Traffic Paths

### Same VLAN

```
A → B directly
(no routing)
```

### Different VLANs

```
A → gateway → firewall rules → B
```

### To the internet

```
A → gateway → NAT → internet
```

### To containers on the same host

Depends on:

- bridge mode  
- host mode  
- VLAN tagging  
- Docker’s internal routing  

Docker networking deserves its own chapter (and therapy), so this document only touches on it lightly.

---

## 7. Troubleshooting — The Practical Checklist

### 1. Device can reach LAN but not internet

Likely:

- incorrect gateway  
- DNS resolution failing  
- NAT misconfigured  

---

### 2. Devices on same VLAN can’t talk

Check:

- subnet mask mismatch  
- ARP entries (`ip neigh`)  
- duplicate IPs  
- Wi-Fi isolation mode  

---

### 3. VLANs can’t route to each other

Check:

- inter-VLAN firewall rules  
- gateway addresses  
- routing table on the firewall  
- missing or incorrect static routes  

---

### 4. Asymmetric routing

Symptoms:

- pings work one way, not the other  
- TCP sessions fail unexpectedly  

Often caused by:

- multiple gateways  
- incorrect static routes  

---

### 5. Gateway unreachable

Check:

- VLAN tagging  
- switchport configuration  
- wrong gateway IP  
- device on guest Wi-Fi instead of LAN  

---

## 8. Worked Examples

### A. Direct LAN Communication

```
10.10.1.20/24 → 10.10.1.50/24
Direct via ARP
```

### B. Inter-VLAN Traffic

```
10.10.10.20 → gateway 10.10.10.1
→ firewall routing
→ 10.10.20.50
```

### C. Mask Mismatch Creates One-Way Visibility

Device A:
```
10.10.1.20/24
```

Device B:
```
10.10.1.50/25
```

Outcome:

- A thinks B is local  
- B thinks A is remote  
- Communication collapses into chaos  

---

## 9. Summary

The LAN + routing model is predictable once you internalise a few rules:

- Same subnet → direct  
- Different subnet → gateway  
- Routing table chooses the best path  
- ARP enables LAN discovery  
- VLANs divide networks cleanly but require routing  
- Most failures trace back to masks, gateways, or firewall rules  

Understand these, and nearly all connectivity problems become solvable with a calm sip of tea.

---

## 10. Appendix

### Useful Commands

```
ip a
ip route
ip neigh
ping <ip>
traceroute <ip>
```

### Routing Table Example

```
default via 10.10.1.1 dev eth0
10.10.1.0/24 dev eth0 proto kernel scope link src 10.10.1.20
```

### Typical VLAN Layout

| VLAN | Subnet          | Purpose        |
|------|-----------------|----------------|
| 10   | 10.10.10.0/24   | Infrastructure |
| 20   | 10.10.20.0/24   | Servers        |
| 30   | 10.10.30.0/24   | Clients        |
| 40   | 10.10.40.0/24   | IoT            |
