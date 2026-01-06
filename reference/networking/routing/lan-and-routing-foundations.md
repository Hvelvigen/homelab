# LAN & Routing — Foundations
A gentle introduction to how traffic actually moves inside and outside your network.

---

## 1. LAN Basics
Devices on the same subnet can talk to each other directly — no routing needed.

Example: 
```
10.10.1.20 ↔ 10.10.1.50
```

If they share the same network (e.g., `10.10.1.0/24`), they communicate freely unless a firewall gets involved.

This is the simplest part of networking:  
same subnet → direct conversation.

---

## 2. Routing Basics
When a device wants to reach something **outside its subnet**, it hands the traffic to the **gateway**.

Your device essentially asks itself:

> “Is the destination inside my network?  
> If yes → send directly.  
> If no → send to the gateway.”

That gateway is usually your router or firewall — it knows how to reach the outside world.

---

## 3. The Routing Table
You can see how your device makes decisions using:

```
ip route
```

Most homelab systems show something like:

```
default via 10.10.1.1
```

Meaning:
- If the system doesn’t have a more specific route  
- It forwards the packet to **10.10.1.1** (the gateway)

It’s basically a list of “if this, send here” rules.

---

## 4. VLANs (A Future Upgrade You’ll Meet Soon)
VLANs split one physical network into **multiple logical networks**, such as:

- LAN  
- Servers  
- IoT  
- Guest  

For now, the only rule you need is:

- **same VLAN = direct communication**  
- **different VLANs = routing required**

Think of VLANs as “subnets that share a cable but not a neighbourhood.”

---

## 5. Summary
LAN and routing fundamentals are straightforward:

- same subnet → direct  
- different subnet → use the gateway  
- routing table decides the path  
- VLANs create multiple subnets on the same hardware  

Once you’re comfortable with this, every other networking concept becomes much easier.
