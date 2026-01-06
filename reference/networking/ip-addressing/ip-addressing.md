# IP Addressing

A practical, detailed, admin-friendly guide to IPv4 addressing in a homelab.  
This document builds on the simple version and gives you clear mental models, correct terminology, and enough depth to operate and troubleshoot confidently.

---

## Purpose of This Guide
This guide explains how IPv4 addressing works in real networks — not exam theory, not abstract maths, just the concepts and behaviours you actually need when building or maintaining a homelab.

It aims to give you:
- a solid understanding of subnets and CIDR
- a clear picture of how devices decide who is “local”
- a practical mental model of DHCP, gateways, and addressing plans
- the ability to troubleshoot IP problems confidently

---

## 1. What an IP Address Actually Is
Every IPv4 address is **32 bits**.  
Those bits are split into two roles:

- **Network portion** → identifies the subnet  
- **Host portion** → identifies the device on that subnet  

Example:
```
10.10.1.42
```

In binary (simplified):
```
00001010.00001010.00000001.00101010
```

When combined with a **subnet mask** or **CIDR suffix**, the device knows which bits represent the network and which represent the host.

---

## 2. Subnets and CIDR (Practical Mental Model)

### CIDR notation
CIDR looks like this:
```
10.10.1.42/24
```
“/24” means:
- the first 24 bits are the network  
- the remaining 8 bits are host addresses  

A /24 holds 256 total addresses:
- **1 network address**
- **254 usable host addresses**
- **1 broadcast address**

### Common homelab subnet sizes
| CIDR | Usable Hosts | Notes |
|------|--------------|-------|
| /24  | 254          | Most common; easy to manage |
| /23  | 510          | Two /24s merged; larger broadcast domain |
| /25  | 126          | Useful for splitting large VLANs |

For most homelabs, **/24** is perfectly sized and predictable.

---

## 3. How Devices Decide Who Is Local
When a device wants to talk to another IP, it checks:

1. **Is the destination in my subnet?**  
2. If yes → it sends traffic directly using **ARP**  
3. If no → it sends traffic to its **gateway**

### ARP in plain English
ARP maps:
```
IP address → MAC address
```

Devices use ARP to discover the physical address of neighbours on the same subnet.  
If ARP breaks (duplicate IPs, stale cache), communication fails.

Useful commands:
```
ip neigh
```

---

## 4. Private IP Address Ranges
These IP blocks are reserved for internal networks and never routed on the public internet:

```
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
```

### Why most homelabs use 10.x.x.x
- far more usable space  
- easy to carve into multiple VLANs  
- simple mental models for clean addressing  

Example:
- 10.10.x.x for infrastructure  
- 10.20.x.x for clients  
- 10.30.x.x for IoT

---

## 5. Gateways, Broadcasts, and Host Addresses

### Gateway
A gateway is a router interface responsible for routing traffic **outside** the local subnet.

Example:
```
10.10.1.1 → the gateway for 10.10.1.0/24
```

### Network address
The first address in a subnet:
```
10.10.1.0/24 → 10.10.1.0
```

### Broadcast address
The last address:
```
10.10.1.0/24 → 10.10.1.255
```

Misusing these breaks things immediately.

---

## 6. DHCP, Static IPs, and Reservations

### DHCP leases
Devices request configuration from the DHCP server:
- IP address
- subnet mask
- gateway
- DNS servers
- hostname

### Reservations
A reservation locks a device’s MAC address to a known IP.  
This keeps addressing predictable without manually configuring every server.

### When static IPs are appropriate
- firewalls  
- hypervisors  
- domain controllers  
- storage systems  
- core switches  
- anything that must be reachable before DHCP is available  

---

## 7. VLANs and IP Addressing
A VLAN is a **layer‑2 broadcast domain**.  
Each VLAN should have **one subnet** assigned.

Mixing subnets inside a VLAN creates:
- ARP confusion  
- routing ambiguity  
- unpredictable behaviour  

Mapping example:
- VLAN 10 → 10.10.10.0/24  
- VLAN 20 → 10.10.20.0/24  
- VLAN 30 → 10.10.30.0/24  

This keeps everything predictable.

---

## 8. Address Planning for a Homelab

### Suggested layout
Infrastructure:
```
10.10.10.0/24  (firewall, hypervisors)
```

Servers:
```
10.10.20.0/24  (docker hosts, DNS, reverse proxy)
```

Clients:
```
10.10.30.0/24  (personal devices)
```

IoT:
```
10.10.40.0/24  (TVs, cameras, smart plugs)
```

This segmentation keeps noise isolated and security simpler.

### Reservation strategy
- reserve all infrastructure devices  
- reserve all servers  
- reserve anything that needs stable DNS entries  
- let general clients use DHCP normally

---

## 9. Common IP Issues and Diagnostics

### 1. Duplicate IPs
Symptoms:
- packet loss  
- intermittent pings  
- ARP table flipping  

Fix:
```
ip neigh
```
Identify MAC address mismatches.

---

### 2. Wrong subnet mask
Symptoms:
- some devices reachable, others not  
- routing behaves inconsistently  

Fix:
- Verify mask on both ends of the connection  
- Ensure all devices in the subnet use the same CIDR  
- Correct the mask and reload the interface/DHCP lease  
- Check that no VLAN is unintentionally mixing subnet sizes  

### 3. Wrong gateway
Symptoms:
- local LAN works  
- internet or inter-VLAN traffic fails  

Fix:
- Verify the default gateway matches the subnet’s actual router  
- Confirm the gateway exists and is reachable (`ping <gateway>`)  
- Ensure no static route is overriding the default route  
- Correct the gateway and renew DHCP/apply config  

### 4. DNS confusion mistaken for IP issues
Symptoms:
- name fails, IP works  
- intermittent success  

Fix:
```
dig <hostname>
dig @dns-server <hostname>
```

---

### 5. ARP issues
Symptoms:
- device reachable then unreachable  
- random latency  

Fix:
```
ip neigh flush all
```

---

### Useful tools
```
ip a
ip route
ip neigh
ping
traceroute
```

---

## 10. Worked Examples

### Example 1: A /24 network
```
Subnet:       10.10.1.0/24
Gateway:      10.10.1.1
Usable hosts: 10.10.1.1 → 10.10.1.254
Broadcast:    10.10.1.255
```

### Example: Small homelab plan
| VLAN | Subnet | Purpose |
|------|--------|---------|
| 10 | 10.10.10.0/24 | Infrastructure |
| 20 | 10.10.20.0/24 | Servers |
| 30 | 10.10.30.0/24 | Clients |
| 40 | 10.10.40.0/24 | IoT |

---

## 11. Summary

IP addresses follow simple rules:
- Subnets define the local range  
- Gateways route traffic out of the subnet  
- VLANs isolate broadcast domains  
- Reservations make addressing predictable  
- Most outages come from masks, gateways, or conflicts  

Master these and most network issues become routine.

---

## 12. Appendix

### Common CIDR sizes
| CIDR | Hosts |
|------|-------|
| /24  | 254 |
| /25  | 126 |
| /23  | 510 |
| /30  | 2 |

### Binary mask table (quick reference)
| Mask | Binary |
|------|--------|
| /24 | 11111111.11111111.11111111.00000000 |
| /25 | 11111111.11111111.11111111.10000000 |
| /23 | 11111111.11111111.11111110.00000000 |

---
