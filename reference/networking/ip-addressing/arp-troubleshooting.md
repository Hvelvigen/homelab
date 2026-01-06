# ARP Troubleshooting
*A practical guide to diagnosing and fixing ARP issues across Linux, BSD, and Windows environments.*

ARP problems are frustrating because they break connectivity in ways that *look* like routing failures, DNS problems, or firewall issues — but none of those things are actually wrong.

This guide walks through systematic ARP troubleshooting with platform-specific commands and real-world scenarios you'll actually encounter.

---

## Repository Location

```
reference/
└── networking/
    └── ip-addressing/
        ├── arp-basics.md
        └── arp-troubleshooting.md   ← you are here
```

---

## Before You Begin

Read **arp-basics.md** first if you're unfamiliar with how ARP works.

This guide assumes:
- you have console or SSH access to affected systems
- you understand basic networking concepts (IP, MAC, subnet)
- you can escalate to administrative privileges when required

---

## 1. Symptoms That Suggest ARP Issues

ARP problems present in specific patterns:

- **Intermittent connectivity** — ping works, then fails, then works again to the same IP
- **One device unreachable** — everything else on the LAN works fine
- **Connectivity breaks after IP/MAC changes** — worked before, stopped after reconfiguration
- **Different devices see different behaviour** — one host connects, another doesn't
- **"Destination host unreachable"** — but routing, DNS, and firewall are all correct
- **Works by IP on some attempts** — not a DNS issue, not a routing issue

If those symptoms match, continue with the diagnostic flow below.

---

## 2. Diagnostic Flow (Universal Approach)

Follow this sequence regardless of platform:

1. **Verify basic connectivity** — can you ping anything on the LAN?
2. **Check the ARP cache** — does the target IP have an entry?
3. **Identify conflicts** — are multiple devices claiming the same IP?
4. **Test ARP refresh** — flush the cache and retry
5. **Verify network scope** — VLAN, subnet, and broadcast domain correct?
6. **Check for interference** — switches, firewalls, or security features blocking ARP?

---

## 3. Platform-Specific Commands

### 3.1 Viewing the ARP Cache

#### Linux
```bash
ip neigh
```

or

```bash
arp -a
```

Example output:
```
10.10.1.1    dev eth0 lladdr aa:11:22:33:44:55 REACHABLE
10.10.1.20   dev eth0 lladdr bb:22:33:44:55:66 STALE
10.10.1.50   dev eth0 lladdr cc:33:44:55:66:77 DELAY
```

States:
- **REACHABLE** — recently validated, currently trusted
- **STALE** — not recently validated but still cached
- **DELAY** — checking validity now
- **PROBE** — actively sending ARP requests
- **FAILED** — unreachable or no response
- **INCOMPLETE** — resolution in progress

---

#### BSD (FreeBSD, OpenBSD, pfSense, OPNsense)
```bash
arp -a
```

or

```bash
ndp -a    # for IPv6 (Neighbor Discovery Protocol)
```

Example output:
```
gateway.local (10.10.1.1) at aa:11:22:33:44:55 on em0 expires in 1197 seconds [ethernet]
server.local (10.10.1.20) at bb:22:33:44:55:66 on em0 permanent [ethernet]
```

---

#### Windows
```powershell
arp -a
```

or

```powershell
Get-NetNeighbor
```

Example output:
```
Internet Address      Physical Address      Type
10.10.1.1             aa-11-22-33-44-55     dynamic
10.10.1.20            bb-22-33-44-55-66     dynamic
10.10.1.50            cc-33-44-55-66-77     static
```

Types:
- **dynamic** — learned via ARP, will expire
- **static** — manually configured, persistent
- **invalid** — entry failed or expired

---

### 3.2 Flushing the ARP Cache

Clearing the cache forces fresh ARP queries.

#### Linux
```bash
sudo ip neigh flush all
```

or for a specific interface:

```bash
sudo ip neigh flush dev eth0
```

or delete a single entry:

```bash
sudo ip neigh del 10.10.1.50 dev eth0
```

---

#### BSD
```bash
sudo arp -d -a
```

or for a specific entry:

```bash
sudo arp -d 10.10.1.50
```

---

#### Windows (Elevated PowerShell or Command Prompt)
```powershell
arp -d
```

or for a specific entry:

```powershell
arp -d 10.10.1.50
```

or using PowerShell:

```powershell
Remove-NetNeighbor -AddressFamily IPv4
```

---

### 3.3 Adding Static ARP Entries

Static entries prevent cache changes and protect critical infrastructure.

#### Linux
```bash
sudo ip neigh add 10.10.1.1 lladdr aa:bb:cc:dd:ee:ff dev eth0
```

To make it permanent across reboots, add to a startup script or network config.

---

#### BSD
```bash
sudo arp -s 10.10.1.1 aa:bb:cc:dd:ee:ff
```

To persist across reboots, add to `/etc/rc.conf`:

```
static_arp_pairs="gateway"
arp_gateway="10.10.1.1 aa:bb:cc:dd:ee:ff"
```

---

#### Windows (Elevated)
```powershell
arp -s 10.10.1.1 aa-bb-cc-dd-ee-ff
```

or using PowerShell:

```powershell
New-NetNeighbor -IPAddress 10.10.1.1 -LinkLayerAddress "aa-bb-cc-dd-ee-ff" -InterfaceAlias "Ethernet"
```

Static entries in Windows persist until removed or the system reboots.

---

### 3.4 Monitoring ARP Activity

Sometimes you need to watch ARP in real time.

#### Linux
```bash
sudo tcpdump -i eth0 arp
```

or

```bash
sudo tcpdump -i eth0 -n arp
```

`-n` skips DNS resolution for clarity.

---

#### BSD
```bash
sudo tcpdump -i em0 arp
```

---

#### Windows
Use **Wireshark** or **Microsoft Message Analyzer** (deprecated but still functional).

Filter: `arp`

Alternatively, PowerShell with elevated privileges:

```powershell
Get-NetNeighbor -AddressFamily IPv4 | Format-Table
```

Run repeatedly to observe changes:

```powershell
while ($true) { Clear-Host; Get-NetNeighbor -AddressFamily IPv4 | Format-Table; Start-Sleep 2 }
```

---

## 4. Common ARP Problems and Solutions

### 4.1 Duplicate IP Address

**Symptoms:**
- intermittent connectivity
- ARP cache flips between different MAC addresses
- logs show ARP replies from multiple sources

**Diagnosis:**

Check if the same IP appears with different MACs:

Linux:
```bash
ip neigh | grep 10.10.1.50
```

BSD:
```bash
arp -a | grep 10.10.1.50
```

Windows:
```powershell
arp -a | findstr 10.10.1.50
```

If you see the IP pointing to different MACs within seconds, you have a duplicate.

**Fix:**

1. Identify which devices claim the IP:
   - check DHCP leases
   - check static IP assignments
   - physically inspect devices if necessary

2. Correct the misconfiguration

3. Flush ARP caches on all affected systems

4. Confirm resolution:

```bash
ping 10.10.1.50
```

Traffic should stabilise.

---

### 4.2 Stale ARP Entry

**Symptoms:**
- device changed IP or MAC but others still use old mapping
- new device unreachable despite correct configuration
- connectivity works after a few minutes (when cache expires naturally)

**Diagnosis:**

Check cache state:

Linux:
```bash
ip neigh show 10.10.1.50
```

BSD:
```bash
arp -a | grep 10.10.1.50
```

Windows:
```powershell
arp -a | findstr 10.10.1.50
```

Compare cached MAC with actual device MAC.

**Fix:**

Flush the specific entry or entire cache:

Linux:
```bash
sudo ip neigh del 10.10.1.50 dev eth0
```

BSD:
```bash
sudo arp -d 10.10.1.50
```

Windows:
```powershell
arp -d 10.10.1.50
```

Then test:

```bash
ping 10.10.1.50
```

ARP will re-query and learn the correct mapping.

---

### 4.3 ARP Not Resolving (No Entry Created)

**Symptoms:**
- ping fails immediately with "Destination Host Unreachable"
- no ARP entry appears in the cache
- tcpdump shows ARP requests but no replies

**Diagnosis:**

Attempt to ping the target, then immediately check the cache:

Linux:
```bash
ping -c 1 10.10.1.50
ip neigh show 10.10.1.50
```

BSD:
```bash
ping -c 1 10.10.1.50
arp -a | grep 10.10.1.50
```

Windows:
```powershell
ping 10.10.1.50
arp -a | findstr 10.10.1.50
```

If no entry appears or shows `INCOMPLETE` / `FAILED`, the device isn't responding to ARP.

**Possible causes:**

1. **Device is offline or unreachable**
   - verify device is powered on
   - check physical connectivity

2. **Wrong subnet**
   - confirm target IP is in the same subnet
   - check subnet masks on both ends

3. **VLAN misconfiguration**
   - verify both devices are on the same VLAN
   - check switch port assignments

4. **Firewall blocking ARP**
   - rare but possible on misconfigured systems
   - check host firewall settings

5. **Network loop or broadcast storm**
   - ARP requests may be flooded or dropped
   - check switch logs for errors

**Fix:**

- confirm network topology
- validate subnet and VLAN configuration
- test from another device on the same segment
- if problem persists, investigate at the switch/network layer

---

### 4.4 ARP Works But Traffic Doesn't Flow

**Symptoms:**
- ARP cache shows correct MAC address
- ARP resolution completes successfully
- but higher-layer connectivity (ping, SSH, HTTP) still fails

**Diagnosis:**

This suggests ARP is working — the problem lies elsewhere.

Check:

1. **Routing**
   ```bash
   ip route
   traceroute 10.10.1.50
   ```

2. **Firewall**
   - verify no firewall rules blocking traffic
   - test with firewall temporarily disabled (if safe)

3. **Service state**
   - is the target service actually running?
   - check listening ports on the target device

**ARP is not the issue here** — move to network, firewall, or application troubleshooting.

---

### 4.5 Gratuitous ARP Causing Conflicts

**Symptoms:**
- device announces itself multiple times on network join
- legitimate devices get overwritten in ARP tables
- connectivity briefly breaks when certain devices connect

**What is Gratuitous ARP?**

A gratuitous ARP is an ARP announcement sent **without being requested**, typically when:
- a device joins the network
- an IP or MAC address changes
- a high-availability (HA) system fails over

It updates other devices' caches proactively.

**Problem:**

If two devices send gratuitous ARPs for the same IP, they compete to "own" that address.

**Diagnosis:**

Monitor ARP traffic:

Linux / BSD:
```bash
sudo tcpdump -i eth0 -n arp
```

Look for repeated unsolicited ARP announcements.

**Fix:**

- identify misconfigured HA or clustering software
- correct duplicate IP assignments
- disable unnecessary gratuitous ARP (if supported by device)

---

### 4.6 ARP Timeout Too Short

**Symptoms:**
- legitimate devices keep disappearing from cache
- repeated ARP requests flood the network
- logs show constant ARP queries for stable devices

**Diagnosis:**

Check ARP timeout values:

Linux:
```bash
ip neigh show
```

Entries should remain `REACHABLE` or `STALE` for reasonable durations (typically minutes).

BSD:
```bash
sysctl net.link.ether.inet.max_age
```

Windows:
```powershell
Get-NetNeighbor | Select-Object IPAddress, State
```

**Fix (Linux):**

Increase timeout (example: 10 minutes):

```bash
sudo sysctl -w net.ipv4.neigh.default.gc_stale_time=600
```

Make permanent:

```bash
echo "net.ipv4.neigh.default.gc_stale_time=600" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

**Fix (BSD):**

```bash
sudo sysctl net.link.ether.inet.max_age=600
```

Add to `/etc/sysctl.conf` to persist.

**Fix (Windows):**

Windows handles this automatically, but you can adjust the interface metric or neighbor cache indirectly through registry settings — generally not recommended unless you know exactly what you're doing.

---

## 5. Advanced Scenarios

### 5.1 ARP and VLANs

**Scenario:**
Devices on the same IP subnet but different VLANs can't see each other.

**Why:**
ARP uses broadcast, and VLANs isolate broadcast domains.

**Fix:**
- ensure devices are on the correct VLAN
- verify VLAN tagging at the switch
- check trunk ports and allowed VLANs
- confirm inter-VLAN routing if required

ARP won't work across VLAN boundaries without routing.

---

### 5.2 ARP and Docker / Containers

**Scenario:**
Container can't reach LAN devices, but host can.

**Why:**
Docker bridge mode uses NAT — containers don't directly participate in LAN ARP.

**Fix:**

Use **macvlan** or **ipvlan** network mode for direct L2 access:

```bash
docker network create -d macvlan \
  --subnet=10.10.1.0/24 \
  --gateway=10.10.1.1 \
  -o parent=eth0 macvlan_net
```

Then run containers on that network:

```bash
docker run --network=macvlan_net ...
```

Containers will now appear as separate devices on the LAN and participate in ARP normally.

---

### 5.3 ARP and Proxmox / Hypervisors

**Scenario:**
VM can't reach LAN even though networking appears correct.

**Why:**
Hypervisor bridge may not be forwarding ARP correctly.

**Fix (Proxmox):**

Check bridge configuration:

```bash
brctl show
```

Ensure VM's network interface is attached to the correct bridge.

Restart networking or reboot the VM.

---

### 5.4 ARP Spoofing / Poisoning

**Scenario:**
Legitimate traffic is redirected or intercepted.

**Why:**
An attacker (or misconfigured device) is sending fake ARP replies.

**Diagnosis:**

Monitor for unexpected ARP changes:

Linux / BSD:
```bash
sudo tcpdump -i eth0 -n arp | grep "is-at"
```

Watch for the same IP announcing different MACs.

**Fix:**

1. **Use static ARP entries** for critical infrastructure:

   ```bash
   sudo ip neigh add 10.10.1.1 lladdr aa:bb:cc:dd:ee:ff dev eth0
   ```

2. **Enable port security** on managed switches:
   - bind MAC addresses to ports
   - limit number of MACs per port

3. **Deploy ARP inspection** (Dynamic ARP Inspection / DAI):
   - available on enterprise grade switches
   - validates ARP packets against DHCP snooping database

4. **Segment the network**:
   - isolate untrusted devices using VLANs
   - limit broadcast domains

---

## 6. Validation Checklist

After troubleshooting, confirm:

- [ ] ARP cache shows correct MAC for target IP
- [ ] No duplicate IPs exist on the network
- [ ] Connectivity is stable (not intermittent)
- [ ] ARP state is `REACHABLE` or equivalent
- [ ] No unexpected gratuitous ARP announcements
- [ ] VLANs and subnets correctly configured
- [ ] Static ARP entries applied where needed

---

## 7. When ARP Isn't the Problem

If you've checked everything above and connectivity still fails, ARP is likely not the issue.

Move on to:

- **Routing** — check routing tables and gateway reachability
- **Firewall** — verify no rules blocking traffic
- **DNS** — ensure name resolution works correctly
- **Service state** — confirm target service is running and listening

ARP failures have specific signatures — if those aren't present, look elsewhere.

---

## 8. Summary

ARP troubleshooting follows a predictable pattern:

1. Check the cache
2. Flush and retry
3. Look for conflicts
4. Verify network scope
5. Test at the switch/VLAN layer

Most ARP issues are:
- duplicate IPs
- stale cache entries
- VLAN misconfigurations

And most fixes involve:
- `ip neigh flush all` / `arp -d`
- correcting IP assignments
- validating network topology

Once you've worked through ARP issues a few times, the symptoms become immediately recognisable — and the fixes become routine.
