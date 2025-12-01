# Networking Essentials

A friendly, practical introduction to the core networking ideas you *actually* need when running a homelab:  
IP addresses, ports, localhost, how devices talk to each other, and why things sometimes don’t.

This guide is intentionally simple and conceptual — perfect for beginners, or anyone who wants a clear mental model before touching firewalls, Docker ports, or service URLs.

---

## 1. What This Guide Teaches

By the end, you’ll understand:

- what an IP address represents  
- how LAN devices communicate  
- what “localhost” truly means  
- what ports are and why services need them  
- what “listening” means  
- how DNS turns names into IPs  
- why something works locally but not from another machine  

---

## 2. Your Local Area Network (LAN)

Your LAN is your private network — your own local internet.

Typical layout:

```
10.10.1.1     # router / firewall (gateway)
10.10.1.20    # Ubuntu or Docker host
10.10.1.30    # NAS
10.10.1.50    # your PC
```

Everything in the same subnet (`10.10.1.x/24`) can talk to everything else unless firewall rules say otherwise.

<details>
<summary><strong>The Details</strong></summary>

### What a LAN Actually Is  
A Local Area Network (LAN) is a group of devices that can communicate directly because they share the same IP network.  
If two devices are in `10.10.1.x/24`, they can exchange traffic without going through the internet or involving external routing.

This makes LAN communication:

- fast  
- predictable  
- completely local to your home/homelab

### Subnets — Why `10.10.1.x/24` Works  
`/24` indicates the **subnet mask**:  

```
255.255.255.0
```

This tells devices:

- the first three octets (`10.10.1`) = the network  
- the last octet (`.x`) = individual devices

So all devices from `10.10.1.1` to `10.10.1.254` are part of the **same broadcast domain** and can see each other directly.

### Common Address Roles  
You can assign addresses however you like, but sensible conventions help avoid confusion, like this example:

- `.1` — gateway/router/firewall  
- `.2–.50` — infrastructure (servers, NAS, VM hosts)  
- `.51–.200` — DHCP pool (laptops, phones, TVs, tablets)  
- `.201–.254` — static/reserved or “special use” addresses  

These aren’t rules — just patterns that keep things tidy.

### ARP (Address Resolution Protocol) — How Devices Find Each Other  
When device A wants to talk to device B on the same LAN, it doesn’t know its MAC address.  
So it asks:

> “Who has 10.10.1.20? Tell 10.10.1.50.”

This ARP request is broadcast across the LAN.  
Device B replies with its hardware (MAC) address.  
Now A can communicate directly.

You don’t need to master ARP, but understanding it explains:

- why devices must be in the same subnet to discover each other  
- why cross-VLAN communication needs routing  
- why misconfigured masks break everything

### Broadcast & Why LANs Have Size Limits  
Some traffic (e.g., ARP) is broadcast to all devices.  
Large broadcast domains become noisy, so homelabs often use VLANs to split networks into clean segments.

But for starter LANs, a `/24` is perfect.

### A Note on VLANs (Optional Future Skill)  
Later, you may introduce segmented networks like:

- LAN  
- IoT  
- Servers  
- Guest  
- Management  

Each becomes its own subnet (e.g., `10.10.20.x/24`, `10.10.30.x/24`, etc.).  
These require routing and firewall rules and potentially specific hardware, but the underlying LAN principles don’t change — you’re just organising them.

</details>

---

## 3. What an IP Address Really Is

An IP address is simply the network version of a postal address.

Example:

```
10.10.1.20
```

- `10.10.1` → the network  
- `.20` → the device on that network  

CIDR format:

```
10.10.1.20/24
```

`/24` means “all addresses from 10.10.1.0 → 10.10.1.255 are part of this network”.

<details>
<summary><strong>The Details</strong></summary>

### What an IP Address Actually Represents
An IP address is a **logical identifier** that tells the network two things:

1. **Which network a device belongs to**  
2. **Which device it is within that network**

It’s the same idea as:

- a street name (the network)  
- a house number (the device)

Both parts together create a complete address.

### IPv4 Structure (the one you use most)
IPv4 addresses use four numbers separated by dots:

```
A.B.C.D
```

Each part can be 0–255.

Example:

```
10.10.1.20
```

- `10.10.1` → the network segment  
- `20` → the device identifier (host)

This format has been around since the 1980s and isn’t going anywhere in homelabs.

### How Devices Know “Which Bit Is Which”
The **CIDR suffix** (`/24` in `10.10.1.20/24`) tells devices **how many bits of the address describe the network** and **how many bits describe individual devices (hosts)**.  
CIDR stands for **Classless Inter-Domain Routing**, and it replaced the old, rigid IP “classes” with a flexible way to define network sizes.

In simple terms:

- A **bigger number** (e.g., `/30`) → **tiny network** (fewer device addresses)  
- A **smaller number** (e.g., `/16`) → **larger network** (more device addresses)

`/24` is common in homelabs because it gives **254 usable addresses**, which is more than enough while still keeping things tidy and predictable.

If we break `/24` down even further:

- first **24 bits** → network  
- last **8 bits** → devices  

Because 8 bits can represent 0–255, a `/24` network contains:

- 256 total addresses  
- `.0` as the *network address*  
- `.255` as the *broadcast address*  
- `.1–.254` usable for devices  

This is why `/24` is the most common home-lab subnet.

### Why IPs Must Be Unique
Within the same network, no two devices can share the same IP address.

If they do:

- ARP replies become inconsistent  
- traffic goes to the wrong device  
- services “randomly” stop responding  
- logs get confusing  
- debugging becomes depressing  

This is why DHCP or DHCP reservations usually handle IP allocations for you.

### Private vs Public IP Ranges
You’ll mostly use **private IP addresses** inside your homelab.  
These ranges are set aside **specifically for internal networks**.  

They are formally reserved by **IANA** (Internet Assigned Numbers Authority) under **RFC 1918**, the standard that defines which IPv4 ranges are safe for private, internal use.  
These blocks are intentionally *not routable on the public internet*, preventing collisions with real-world hosts and giving homes, labs, and organisations predictable address space.

The three private IPv4 ranges are:

```
10.0.0.0/8        # very large private range (~16.7 million addresses)
172.16.0.0/12     # medium private range (1,048,576 addresses)
192.168.0.0/16    # common home-router range (65,536 addresses)
```

These are the **only IPv4 ranges officially reserved for private use**.  
Everything else is considered public and must be globally unique.

In practice:

- **10.x.x.x** → chosen by many homelabbers because it allows lots of tidy VLAN subnets  
- **172.16–31.x.x** → used often in corporate networks  
- **192.168.x.x** → typical home router default

Functionally they behave the same — the choice is mainly about scale, organisation, and convention.

Your firewall/router performs **NAT (Network Address Translation)**, which converts your private addresses into a public one when traffic leaves your network — and reverses the process for replies coming back.

### Why You See 127.0.0.1 Sometimes
`127.0.0.1` is a **loopback** address — also called `localhost`.

It always refers to the machine you’re currently using.  
It never refers to another machine, even if that other machine is also using 127.0.0.1 (because they all do).

### IPv6: The Briefest Possible Mention
IPv6 exists, and looks like this:

```
fe80::4c3b:2ff:fe90:ae52
```

It doesn’t replace IPv4 — both can run side-by-side.  
It solves global address exhaustion and does clever things.  
For homelab basics, you can safely ignore IPv6 until you want to explore it.

### IP Addressing Mistakes Beginners Often Make
- Thinking `localhost` = “my server”  
- Forgetting that every device needs its own IP  
- Mixing up `10.10.1.x` and `10.10.2.x`  
- Using the wrong subnet mask  
- Forgetting to update DNS after IP changes  
- Assuming a service bound to 127.0.0.1 will accept LAN traffic (it won’t)

### One-sentence summary
> An IP address tells the network *where a device lives* and the CIDR mask tells the device *who counts as a neighbour*.

</details>

---

## 4. Gateway (How You Reach Other Networks)

Your gateway is the device that knows how to get traffic *out* of the LAN.

Usually your router or OPNsense firewall:

```
Gateway: 10.10.1.1
```

If the gateway is unreachable:

- internet access fails  
- DNS may not work  
- updates and mirror checks fail  

<details>
<summary><strong>The Details</strong></summary>

### What the Gateway Actually Does
Inside a LAN, devices talk directly to each other.  
The moment traffic needs to leave the subnet — the internet, another VLAN, a VPN, anything not `10.10.1.x` — your device needs help.

That helper is the **gateway**.

Your device effectively asks:

> “Is the destination inside my subnet?  
> If yes → send traffic directly.  
> If no → hand it to the gateway.”

The gateway then **routes** the traffic to wherever it needs to go.

### How Devices Decide Whether to Use the Gateway
Each device uses two configuration values:

1. **Its own IP address**  
2. **Its subnet mask or CIDR (/24)**  

Together they tell the device:

- “Everything in 10.10.1.x is local.”  
- “Everything else needs the gateway.”

If either value is wrong, routing breaks.

### How the Gateway Handles Internet Traffic (NAT)
Your LAN uses private IPs; the internet uses public IPs.  
Your firewall/router performs **Network Address Translation (NAT)**:

- Outbound traffic → rewritten to your public IP  
- Inbound replies → mapped back to the original internal device  

This is how many devices share a single internet connection without conflict.

### The Gateway and Multiple VLANs
In a more advanced homelab, the gateway is responsible for:

- routing LAN ↔ IoT  
- routing LAN ↔ Server networks  
- routing management/trusted networks  
- enforcing firewall rules between them  

If routing rules are missing or blocked, cross-VLAN access fails even when everything local works.

### Quick Diagnostic: “Is the Gateway Working?”
From any device:

```bash
ping -c 4 <gateway-ip>
```

If this fails:

- the device’s network config is wrong, **or**  
- the gateway is offline, **or**  
- the switch/VLAN path isn’t correct  

If the ping works but the internet doesn’t — look at DNS or firewall rules.

### One-sentence summary
> The gateway is the device that handles anything your LAN can’t reach on its own — the internet, other subnets, and everything beyond.

</details>

---

## 5. Subnets (The Briefest Possible Explanation)

Subnets split networks into predictable chunks.

`/24` is the most common and easiest:

- 256 addresses  
- `.1` usually the gateway  
- `.2`–`.254` usable for devices  

That’s enough for homelab basics.

<details>
<summary><strong>The Details</strong></summary>

### What a Subnet Actually Is  
A subnet is a way of dividing an IP range into smaller, organised sections.  
It defines two things:

1. **Which devices are considered “local” to each other**  
2. **Which addresses belong to this group vs another group**

Think of it as putting devices into clearly labelled “rooms” so they know who their neighbours are.

### Why Subnets Exist  
Without subnets, every device would try to talk directly to every other device everywhere — which isn’t possible or scalable.  
Subnets:

- keep broadcast traffic contained  
- provide structure and organisation  
- allow networks to be separated for security  
- let routers decide when to route traffic between networks

### Why Homelabs Commonly Use `/24`  
- It matches how most home routers already behave  
- It’s big enough for almost any home setup  
- It keeps broadcast domains reasonably small  
- It divides cleanly when planning VLANs  
- Tutorials and examples commonly assume it

Unless you’re deploying hundreds of devices or segmenting into dozens of subnets, `/24` simply “just works”.

### Other Subnet Sizes (Just for Context)
You’ll occasionally see:

- `/16` → **very large** network (65,536 addresses)  
- `/20` → 4 × `/24` blocks combined  
- `/30` → used for router-to-router links (only 2 usable addresses)

You don’t need these now, but knowing they exist helps later.

### One-sentence summary
> A subnet defines who is local, who isn’t, and whether a device should talk directly or send traffic through the gateway.

</details>

---

## 6. DNS (Turning Names into IPs)

DNS converts readable names → machine-friendly IPs.

Examples:

```
google.com              → 142.250.x.x
somecooldomain.net      → 67.16.x.x
unifi.local             → 10.10.1.21
```

Useful tools:

```bash
nslookup <name>
dig <name>
getent hosts <name>
```

If DNS breaks, many services appear “down” when they’re not.

<details>
<summary><strong>The Details</strong></summary>

### What DNS Actually Does  
Computers communicate using IP addresses, not names.  
DNS (Domain Name System) is simply the phonebook that converts:

```
name → IP address
```

So instead of remembering:

```
10.10.1.21
```

you can type:

```
unifi.local
```

DNS makes things usable for humans without changing how the network works underneath.

### Why DNS Matters in a Homelab  
A surprising number of things rely on DNS:

- browsers  
- package managers  
- Docker pulling images  
- time synchronisation  
- internal dashboards  
- system services  

When DNS breaks, these tools can’t find the systems they depend on — so everything *looks* broken even when the underlying service is fine.

### How DNS Resolves a Name  
In simple steps:

1. Your device checks its local DNS cache  
2. If not found, it asks the configured DNS server  
3. That DNS server either knows the answer or forwards the query  
4. The response returns with the matching IP  
5. Your device remembers it temporarily (caching)

This is why changing IPs without updating DNS causes chaos.

### Local DNS vs Public DNS  
You can have both:

- **Local DNS** resolves internal names like:

  ```
  unifi.local → 10.10.1.21
  ```

- **Public DNS** resolves internet names:

  ```
  google.com
  ubuntu.com
  github.com
  ```

If local DNS breaks, internal resources become unreachable by name.  
If public DNS breaks, the internet appears “down”.

### How `.local` Works  
Names ending in `.local` are usually handled via **mDNS** (multicast DNS) —  
Apple devices, Linux, and many IoT devices use this for discovery.

Example:

```
unifi.local → 10.10.1.21
```

This does not require a DNS server, but it requires compatible devices on the same LAN.

### DNS Caching  
DNS results are cached for performance:

- system cache  
- browser cache  
- application cache  

If you change a device’s IP but caching hasn’t expired yet, you may hit the wrong system.  
This can cause the “Why is it still pointing to the old IP?” moment.

### Useful Troubleshooting Commands  

- `nslookup` → quick check of who resolves a name and to what  
- `dig` → detailed breakdown (TTL, authoritative server, etc.)  
- `getent hosts` → shows how the **local resolver** interprets the name (includes hosts file, DNS, mDNS)

If these disagree with each other, something is misconfigured.

### Hosts File — The Manual Override  
Every device has a small file that can force name → IP mappings:

```
/etc/hosts                           # Linux
C:\Windows\System32\drivers\etc\hosts   # Windows
```

Example entry:

```
10.10.1.1 gateway.homelab.local
```

This bypasses DNS entirely — great for testing, dangerous if forgotten.

### One-sentence summary  
> DNS is the system that translates names into IP addresses so humans don’t need to memorise numbers — and when it breaks, almost everything feels offline even if it isn’t.

</details>

---

## 7. What “localhost” Actually Means

`localhost` = **this machine only**.

It always resolves to:

```
127.0.0.1    # IPv4
::1          # IPv6
```

So:

- “localhost on the server” = the server  
- “localhost on your PC” = your PC  
- “localhost on your phone” = your phone  

This explains the classic confusion:

> “It works on localhost but not from my laptop!”

Because localhost is always self‑referential.

<details>
<summary><strong>The Details</strong></summary>

### What `localhost` Really Represents  
`localhost` is a special, reserved hostname that always means **“the device I am currently using.”**  
It doesn’t depend on your network, your IP address, your Wi-Fi, or anything external.

It is a built-in loopback interface used for:

- internal testing  
- services talking to themselves  
- software that needs a network connection but doesn’t leave the machine  

This is why every device in the world can use `127.0.0.1` without conflict — it never travels across the network.

### How Localhost Bypasses the Network  
When you connect to:

```
http://localhost:8080
```

Your device does **not** send packets to the LAN.  
It doesn’t ARP.  
It doesn’t contact the gateway.  
It doesn’t touch your firewall.

It simply loops the traffic back into the device’s own network stack.

This is why:

- localhost works even with no network connection  
- you can access local apps during outages  
- testing locally doesn’t test LAN accessibility  

### Why Localhost Does Not Work From Other Devices  
A very common beginner misunderstanding:

```
localhost != the server you’re working on
```

If you run a service on a server and access it via:

```
http://localhost:8080
```

That is **only** valid from the server itself.

If you try that URL from your laptop, your laptop will look for a service running on *itself*, not on the server.

To access the server from another device, you must use:

- its LAN IP (e.g., `10.10.1.20`)  
- or its hostname (if DNS is configured)  

### Services Bound to Localhost  
Some services intentionally listen only on the loopback interface:

```
127.0.0.1:8080
```

This means:

- they are accessible only from the device itself  
- they do not accept connections from the LAN  
- external access will always fail

Common reasons:

- security: local apps shouldn’t be exposed  
- development/testing environments  
- misconfigured Docker or application settings  

This is why you sometimes see:

> “It works on the server but nothing else can reach it.”

### Why `localhost` Is Still Useful  
Despite the confusion it can cause, localhost is invaluable for:

- quickly testing applications  
- running admin tools securely  
- checking that a service is actually running  
- verifying port mappings before exposing services to the LAN  

### One-sentence summary  
> `localhost` always means “me” — not your server, not your network, not another device — just the machine you’re currently using.

</details>

---

## 8. Ports — The Doorways to Services

An IP identifies *a device*.  
A port identifies *a service on that device*.

Examples:

```
22      SSH
80      HTTP
443     HTTPS
2022    custom SSH (commonly used in homelab hardening)
3000    dashboards
8080    web tools
```

To reach a service:

```
http://10.10.1.20:8080
```

This means:

- talk to IP `10.10.1.20`  
- access port `8080`  
- reach the service listening there  

<details>
<summary><strong>The Details</strong></summary>

### Why Ports Exist  
A single device can run *many* services at the same time:

- a web server  
- SSH  
- a database  
- a monitoring tool  
- Docker containers  
- dashboards  
- internal APIs  

Ports allow all of these to coexist without clashing by giving each service its own “doorway”.

Think of an IP address as a building, and ports as the numbered doors leading to different rooms.

### Port Numbers Are Standardised (Mostly)  
Some ports are globally recognised:

- `22`   → SSH  
- `53`   → DNS  
- `80`   → HTTP  
- `443`  → HTTPS  
- `3306` → MySQL  
- `5432` → PostgreSQL  

Others are up to you or the application:

- `3000`, `8080`, `9000+` → dashboards, admin tools, web apps  
- `2022` → common alternative SSH port for homelab hardening  

The port number doesn’t change the service — it only defines *where* it listens.

### What “Listening” Actually Means  
A service only receives traffic if it is **listening** on a port.

You can check this with:

```bash
ss -tuln
```

If you see:

```
0.0.0.0:8080
```

the service is waiting for connections on **all** interfaces.  
If you see:

```
127.0.0.1:8080
```

it is listening **only on localhost** — meaning other devices cannot reach it.

### Docker and Port Mapping  
Docker introduces an important concept:

```
-p <host-port>:<container-port>
```

Example:

```
-p 8080:80
```

Means:

- The **host listens on 8080**  
- Traffic is forwarded into the container’s **port 80**  
- You access it at:  

```
http://10.10.1.20:8080/
```

This mapping is often the reason local apps work on one port but containers behave differently.

### Firewalls and Ports  
Even if a service is running, your firewall may block the port.

Check UFW status with:

```bash
sudo ufw status
```

If a port isn’t listed as **ALLOW** and the firewall is enabled, devices on the LAN can’t reach it.

### Common Port Conflicts  
If two apps try to use the same port, only one can succeed.  
Typical symptoms:

- “Port already in use”  
- Service fails to start  
- Docker container stuck in restart loop  

Changing the port or freeing the old one resolves this quickly.

### One-sentence summary  
> Ports let a single device run many services at once — they’re the numbered doorways that network traffic uses to reach the right application.

</details>

---

## 9. Listening Addresses: 0.0.0.0 vs 127.0.0.1 vs LAN IP

A service can listen on different network interfaces, and this determines **who can access it**.

### `127.0.0.1`  
Service listens only to itself — *local-only*.

### `<LAN IP>` (e.g., `10.10.1.20`)  
Service listens only on the LAN interface.

### `0.0.0.0`  
Service listens on all interfaces (localhost + LAN).

Example:

```
0.0.0.0:8080
```

→ accessible from the entire LAN (subject to firewall rules).

<details>
<summary><strong>The Details</strong></summary>

### Why Listening Addresses Matter  
Even when a service is running, it might not be reachable from other devices.  
This usually isn’t a firewall issue — it’s because the service is only listening on the **wrong interface**.

Listening addresses determine **who can access the service**:

- just the local machine  
- only devices on the LAN  
- everything on every interface  

Understanding this removes most of the “it works here but not there” confusion.

### `127.0.0.1` — Localhost Only  
When a service binds to:

```
127.0.0.1:<port>
```

It means:

- only the **local machine** can access it  
- nothing else on the LAN can reach it  
- Docker containers typically cannot reach it unless bridged  
- remote access always fails  

This is the most restrictive binding.

Used intentionally for:

- admin tools  
- health endpoints  
- development environments  
- services not meant to be exposed

### `<LAN IP>` — Specific Network Interface  
Example:

```
10.10.1.20:<port>
```

This means the service only listens on the server’s LAN interface.

Implications:

- accessible from devices on the same LAN  
- *not* accessible via `localhost` unless using the LAN IP specifically  
- *not* accessible from other VLANs unless routing is configured  

This is a good choice when you want a service available inside the LAN but not exposed elsewhere.

### `0.0.0.0` — All Interfaces  
Example:

```
0.0.0.0:8080
```

This means the service is listening on:

- localhost (127.0.0.1)  
- the LAN interface (e.g., 10.10.1.20)  
- any additional network interfaces (VLANs, bridges, Docker networks)  

This is the broadest binding and is extremely common in Docker containers, which default to exposing published ports on all interfaces.

Access depends entirely on **firewall rules**.

### How to Check What a Service Is Actually Listening On  
Run:

```bash
ss -tuln | grep <port>
```

Interpretation:

- `127.0.0.1:8080` → local-only  
- `10.10.1.20:8080` → LAN-only  
- `0.0.0.0:8080` → all interfaces  

This single command resolves a huge portion of beginner networking problems.

### One-sentence summary  
> Listening addresses decide *who* can reach a service — only the local machine, devices on the LAN, or anything connected to any interface.

</details>

---

## 10. Checking What’s Running

### Ports currently in use:

```bash
ss -tuln
```

### Ports + process names:

```bash
ss -tulnp
```

### Check a specific port:

```bash
sudo ss -lptn | grep <port>
```

If nothing is listening → nothing will respond.

<details>
<summary><strong>The Details</strong></summary>

### Why Checking Listening Ports Matters  
When a service “isn’t working,” the first question is:

> *“Is anything actually listening on the port I’m trying to reach?”*

If nothing is listening, no amount of firewall changes, DNS tweaks, or browser refreshing will help — the service simply has nothing open to receive traffic.

This section gives you the tools to confirm what *is* and *isn’t* listening.

### Understanding the `ss` Command  
`ss` stands for **socket statistics**.  
It replaced the older `netstat` command and is now the standard way to inspect:

- open ports  
- listening sockets  
- bound interfaces  
- owning processes (when run with sudo)  

You mostly care about **LISTEN** entries, which show what services are waiting for connections.

### Reading the Output  
Example:

```
LISTEN 0 128 0.0.0.0:8080 0.0.0.0:*
```

Breakdown:

- **LISTEN** → waiting for incoming connections  
- **0.0.0.0:8080** → service is listening on port 8080 on *all* interfaces  
- `*:*` → it will accept incoming requests from anywhere  

If you instead see:

```
127.0.0.1:8080
```

→ the service is *local-only* and won’t accept LAN connections.

Or:

```
10.10.1.20:8080
```

→ LAN-only, not reachable via localhost unless using that exact IP.

This ties directly into the previous section on listening addresses.

### Using `ss -tuln`  
Shows **listening TCP and UDP ports** without showing which process owns them.

Useful for quick scans, like:

- “Is anything actually listening on port 80?”  
- “Did the service start?”  
- “Why is this port still in use?”  

### Using `ss -tulnp`  
Adds **process information**:

```
LISTEN 0 128 0.0.0.0:22 0.0.0.0:* users:(("sshd",pid=816,fd=3))
```

Now you can see:

- which binary owns the port (`sshd`)  
- the process ID (`pid=816`)  
- the file descriptor (`fd=3`)  

This is invaluable when diagnosing:

- port conflicts  
- stuck or zombie services  
- accidental processes occupying important ports  

### Checking a Specific Port  
Filtering with grep focuses your view:

```bash
sudo ss -lptn | grep 8080
```

If this returns nothing:

- nothing is listening  
- the service may have crashed  
- the service may not have started  
- the service may be bound to IPv6 only  
- the service may have bound to a different port than expected  

This instantly narrows the problem space.

### One-sentence summary  
> Checking what’s listening on which port quickly reveals whether a service is reachable, misconfigured, or not running at all.

</details>

---

## 11. Putting It Together: Troubleshooting Basics

When something “doesn’t work”, these checks cover most beginner homelab issues.

### 1. Can I reach the device at all?

```bash
ping <server-ip>
```

If this fails → IP, gateway, subnet, or VLAN problem.

### 2. Is DNS lying to me?

```bash
nslookup <name>
```

- Try the IP directly:  
  - If IP works but name doesn’t → DNS issue.  
  - If neither work → network or service issue.

### 3. Is anything listening on the port?

```bash
ss -tuln | grep <port>
```

If nothing shows up:

- the service isn’t running  
- or it bound to a different port  
- or it bound to IPv6 only

### 4. Who can actually reach the service?

Use `ss` to check the listening address:

- `127.0.0.1:<port>` → only the local machine  
- `<LAN IP>:<port>` → LAN-only  
- `0.0.0.0:<port>` → all interfaces (then check firewall)

Common patterns:

- Works on the server, not your laptop → likely bound to `127.0.0.1`.  
- Works on LAN, not other VLANs → bound only to the LAN IP, or routing/firewall incomplete.

### 5. Is the firewall blocking it?

```bash
sudo ufw status
```

If the port isn’t listed as **ALLOW** and UFW is active, LAN devices can’t reach it.

### 6. Does the gateway work?

```bash
ping <gateway-ip>
```

If local devices can talk to each other but nothing reaches the internet, suspect:

- gateway config  
- upstream connectivity  
- or DNS

### 7. Do subnet masks match?

Symptoms of mismatch:

- Device A can reach B, but not the other way round  
- Some things ping, others don’t, on “the same LAN”  
- Services visible from one device but not another  

Check both devices’ IP + subnet.  
Mismatch almost always causes these oddities.

---

> When in doubt: **IP → ping → DNS → port → listening address → firewall.**  
> That simple flow will debug most basic networking issues.

---

## 12. Mini Exercises

These short tasks help reinforce the networking basics you've just learned.

1. **Find your server on the LAN:**  
   Run `ping <server-ip>` from your PC.  
   Does it respond?

2. **Check DNS resolution:**  
   Try `nslookup google.com` and `nslookup <your-hostname>`.  
   Compare public vs local lookups.

3. **Inspect what’s listening:**  
   On your server, run `ss -tuln`.  
   Identify one TCP and one UDP service.

4. **Test a specific port:**  
   Run `sudo ss -lptn | grep 22` to confirm SSH is listening.  
   Then try a port you *expect* to be open (e.g., 8080).

5. **Check listening addresses:**  
   Run `ss -tuln | grep <port>`.  
   Is it bound to `127.0.0.1`, your LAN IP, or `0.0.0.0`?

6. **Access a service by IP vs name:**  
   Visit `http://<server-ip>:<port>`  
   Then try `http://hostname:<port>`.  
   If one works but the other doesn’t → DNS issue.

7. **Safe break/fix exercise:**  
   Temporarily change your DNS server to something invalid.  
   Try browsing.  
   Observe what works and what fails.

---

## 13. Summary

The essentials in one place:

- **LAN:** Devices in the same subnet talk directly.  
- **IP Address:** Identifies where a device lives on the network.  
- **Subnet/CIDR:** Defines who counts as a neighbour.  
- **Gateway:** Handles anything outside your local subnet.  
- **DNS:** Converts names into IPs.  
- **localhost:** Always the machine you're using — never another device.  
- **Ports:** Numbered doorways to services.  
- **Listening addresses:** Decide who can reach a service (local-only, LAN, or all).  
- **`ss`:** The fastest way to see what’s running and where it’s listening.

> When something doesn’t work:  
> **IP → ping → DNS → port → listening address → firewall.**  
> This simple flow solves most beginner homelab networking issues.
