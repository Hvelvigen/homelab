# ARP Basics
*How devices actually find each other on a LAN — and why it sometimes stops working.*

ARP (Address Resolution Protocol) is one of those things that works so reliably you forget it exists — right up until it doesn't, and suddenly nothing on your network can talk to anything else.

This guide explains what ARP does, why it matters, and how to recognise when it's misbehaving.

---

## 1. What ARP Actually Does

IP addresses are how *you* think about network locations.  
MAC addresses are how *network hardware* actually delivers packets.

ARP is the translator between the two.

When a device wants to send data to another device on the same LAN, it needs to know:
- the destination IP address (you provide this)
- the destination MAC address (ARP figures this out)

Without ARP, your perfectly valid IP traffic would have nowhere to go — because the network card doesn't understand IP addresses, only MAC addresses.

---

## 2. How ARP Works (The Neighbourhood Shout)

ARP operates like a polite but persistent neighbourhood announcement system.

Here's the full exchange:

1. Device A wants to talk to `10.10.1.50` but doesn't know its MAC address
2. Device A broadcasts to the entire LAN:  
   *"Who has 10.10.1.50? Please reply to [Device A's MAC]"*
3. Every device on the LAN receives the broadcast
4. The device with `10.10.1.50` responds directly:  
   *"That's me. My MAC address is aa:bb:cc:dd:ee:ff"*
5. Device A caches this information and sends the actual traffic

The broadcast is visible to everyone.  
The reply is direct.  
The result is cached so the question doesn't need asking again for a while.

It's efficient — until something goes wrong.

---

## 3. The ARP Cache (Where Things Get Remembered)

Once a device learns a MAC address, it stores the mapping temporarily in its **ARP cache** (also called the ARP table).

This avoids flooding the network with repeated "Who has...?" broadcasts.

View your ARP cache:
```bash
ip neigh
```

or on some systems:
```bash
arp -a
```

Example output:
```
10.10.1.1    lladdr  aa:11:22:33:44:55  REACHABLE
10.10.1.20   lladdr  bb:22:33:44:55:66  STALE
10.10.1.50   lladdr  cc:33:44:55:66:77  REACHABLE
```

States you'll see:
- **REACHABLE** — recently confirmed, currently trusted
- **STALE** — hasn't been validated recently but still in cache
- **DELAY** — checking if the entry is still valid
- **FAILED** — couldn't reach the device

Entries expire after a timeout (usually a few minutes), at which point ARP will query again if needed.

---

## 4. Why ARP Breaks (Common Failure Modes)

ARP is simple and reliable — which is exactly why failures are so disruptive when they happen.

### 4.1 Duplicate IP Addresses

If two devices claim the same IP:
- ARP responses conflict
- the cache flips between MAC addresses
- communication becomes intermittent or fails entirely
- logs fill with confused entries

The ARP table can't settle on a single MAC address because two devices keep announcing conflicting mappings.

Symptoms:
- `ping` works sometimes, fails other times
- SSH drops randomly
- web pages half-load then time out

Fix:
- identify which devices share the IP
- correct the configuration
- flush ARP caches across affected devices

---

### 4.2 Stale ARP Entries

A device changes IP or MAC address, but other devices still have the old mapping cached.

Result:
- traffic is sent to the wrong destination
- the intended device appears unreachable
- everything *looks* correct from the sender's perspective

Fix:
```bash
sudo ip neigh flush all
```

This forces devices to re-query and rebuild correct mappings.

---

### 4.3 Misconfigured Switches or VLANs

ARP relies on broadcasts reaching all devices in the same subnet.

If VLANs are misconfigured:
- broadcasts don't propagate correctly
- devices on the same logical subnet can't find each other
- DHCP may fail
- gateways become unreachable

This one often presents as "networking is fundamentally broken" — which is accurate, just not helpful.

---

### 4.4 ARP Storms

Rare but memorable.

Caused by:
- network loops
- misconfigured broadcast domains
- faulty devices flooding ARP requests

Symptoms:
- entire network becomes unusable
- switches saturate
- performance collapses instantly

Fix:
- identify and disable the loop or faulty device
- verify spanning tree protocol (STP) is working
- flush ARP tables once stability returns

---

## 5. ARP Security Considerations

ARP has no authentication.  
Any device can claim to be any IP.  
This makes ARP spoofing trivial.

### ARP Spoofing (Simplified)

An attacker broadcasts:
> "10.10.1.1 is at [attacker's MAC]"

Other devices update their cache and now send gateway traffic to the attacker instead of the real gateway.

This enables:
- traffic interception
- man-in-the-middle attacks
- denial of service

Defences:
- static ARP entries for critical infrastructure
- network segmentation
- monitoring for unexpected ARP changes
- switch port security features

For most homelabs, basic awareness is sufficient — unless you're practising pentesting, in which case you likely already know this section rather well.

---

## 6. Useful ARP Commands

### View the ARP cache
```bash
ip neigh
```

or
```bash
arp -a
```

---

### Flush the ARP cache
```bash
sudo ip neigh flush all
```

Forces all entries to be re-queried.

---

### Add a static ARP entry
```bash
sudo ip neigh add 10.10.1.1 lladdr aa:bb:cc:dd:ee:ff dev eth0
```

Useful for gateways or critical infrastructure that shouldn't change.

---

### Delete a specific entry
```bash
sudo ip neigh del 10.10.1.50 dev eth0
```

Removes one mapping without flushing everything.

---

## 7. When to Suspect ARP

ARP issues present in specific ways:

- local LAN works, but one device is unreachable
- intermittent connectivity that resolves itself after a few minutes
- sudden loss of connectivity after IP changes
- unexplained "destination host unreachable" errors
- traffic works from some devices but not others (suggesting stale cache entries)

If routing, DNS, and firewalls are all correct but something still won't connect — check ARP.

---

## 8. ARP in Docker and VMs

Containers and VMs add extra ARP layers.

### Docker
- containers share the host's ARP table (bridge mode)
- macvlan networks generate their own ARP entries
- conflicts can occur if container IPs overlap with LAN devices

### VMs
- hypervisors handle ARP translation between virtual and physical networks
- promiscuous mode settings affect ARP visibility
- bridged VMs appear as separate devices on the LAN

If containers or VMs can't reach the LAN but the host can, ARP is often involved — usually because the network mode or bridge configuration isn't what you think it is.

---

## 9. Summary

ARP is simple:
- translates IP addresses to MAC addresses
- uses broadcasts and caching
- works transparently when healthy
- breaks in predictable ways

Most ARP problems are:
- duplicate IPs
- stale cache entries
- VLAN or switch misconfigurations

And most fixes are:
```bash
sudo ip neigh flush all
```

Once you understand ARP, a whole category of "networking is broken" moments become one of those rare "ah, it's just ARP being ARP" moments — which is far more manageable.

And on that note, if flushing the ARP cache fixes a problem, it's worth asking *why* the cache was wrong in the first place — because that's usually where the real issue (Root Cause) lives.
