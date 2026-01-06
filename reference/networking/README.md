# Networking Reference

This directory provides practical, actionable networking guides for homelabs.  
Everything here is focused on what you actually need to understand when building, troubleshooting, or expanding your lab. 

---

## What's Here

This section covers: 

- **IP Addressing** — how IP addresses work and why they matter  
- **LAN & Routing** — how devices communicate and when traffic needs routing  
- **Ports & Services** — why your service works locally but not remotely (and how to fix it)  
- **Networking Essentials** — the foundational concepts that tie everything together  

---

## How to Use This Section

Each topic comes in two versions:

### **Simple Version**
Start here if: 
- You're new to networking  
- You want a quick mental model without drowning in detail  
- You just need to understand the core idea  

These guides teach you *why* things work the way they do, not just commands to memorise.

### **Full Version**
Read this if:
- You've grasped the simple version and want more depth  
- You need to diagnose problems confidently  
- You want to understand edge cases and nuances  

These guides give you the vocabulary and patterns to troubleshoot in minutes rather than hours.

---

## Suggested Reading Order

If you're starting from scratch: 

1. **Networking Essentials** (`networking-essentials.md`)  
   *The foundation.  Read this first — it's genuinely worth 20 minutes.*

2. **IP Addressing — Simple** (`ip-addressing-simple.md`)  
   *Understand what IP addresses are and why CIDR matters.*

3. **LAN & Routing — Simple** (`lan-and-routing-simple.md`)  
   *How devices decide who to talk to.*

4. **Ports & Services — Simple** (`ports-and-services-simple.md`)  
   *Why services work here but not there.*

Then, when you're comfortable (or when you hit a problem):

5. **IP Addressing — Full Version** (`ip-addressing.md`)  
   *Subnets, gateways, and the mental models for troubleshooting.*

6. **LAN & Routing — Full Version** (`lan-and-routing.md`)  
   *Routing tables, VLAN basics, and real failure modes.*

7. **Ports & Services — Full Version** (`ports-and-services.md`)  
   *Binding addresses, firewall interaction, and Docker networking.*

---

## Common Questions (Quick Answers)

### "What's the difference between the simple and full versions?"
**Simple** teaches the concept and mental model.   
**Full** gives you vocabulary, edge cases, and troubleshooting patterns.

You don't *need* the full versions to run a homelab, but they're invaluable when things go wrong.

---

### "Do I need to understand this stuff if I'm just running Docker?"
**Short answer:** Yes.   
**Longer answer:** Docker abstracts some networking complexity, but understanding ports, listening addresses, and IP basics saves you hours.   
When something doesn't work, you'll know where to look.

---

### "Which IP range should I use:  10.x.x.x, 172.16.x.x, or 192.168.x.x?"
All three are valid private ranges.   
**Most homelabs use 10.x. x.x** because it offers the most usable space and is easy to split into multiple VLANs without overthinking it.

See **IP Addressing — Full Version**, section 4 for more detail.

---

## How These Guides Fit into Your Homelab Journey

- **Week 1:** Read Networking Essentials + Simple versions.  Your homelab mostly works. 
- **Week 4:** Read Full versions.  You now diagnose problems instead of reinstalling.
- **Month 3:** You reference the specific parts you need when something unusual happens.
- **Year 2:** You're casually explaining these concepts to someone else.

That's the goal. 

---

*Everything here assumes you're building a practical homelab, not studying for a networking exam.   
If you need to pass the CCNA, these guides are a foundation, **not** a replacement for structured study.*
