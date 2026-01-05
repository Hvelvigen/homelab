This is a placeholder document and will be separated into areas of concern.

# DNS — The Theory Version
A deep, technical exploration of how DNS actually works, why it behaves the way it does, and where real-world implementations diverge from simplified explanations.

This document assumes familiarity with:
- DNS — The Simple Version
- DNS — Full Version (Practical Admin Depth)

It focuses on protocol mechanics, architectural intent, and implementation nuance rather than troubleshooting shortcuts.

---

## 0. Scope and Intent

**Purpose of this document**

Explain DNS as:
- a protocol family
- a distributed database
- a trust and delegation system
- an eventually-consistent, cache-heavy control plane

Clarify what DNS *is not*:
- a directory service
- a real-time system
- a strongly consistent database

**Audience**
- homelab builders
- system administrators
- infrastructure engineers
- anyone who has asked “why does DNS behave like this?”

---

## 1. DNS as a Distributed System

At its core, DNS is a **globally distributed, hierarchical database**. There is no single server, organisation, or system that holds “the DNS”. Instead, responsibility for DNS data is **deliberately fragmented** across millions of independently operated servers, each authoritative only for the portion of the namespace it has been delegated. This design is intentional: it avoids central points of failure, distributes operational load, and allows the system to scale to internet size.

DNS has **no *single* global source of truth**. Each zone is authoritative only for itself, and knowledge of that zone is not automatically synchronised everywhere else. Other systems learn about it through queries, caching, and delegation. This means DNS behaves more like a *published dataset* than a live database. Updates are announced, cached, and gradually observed — not pushed instantly to all consumers.

**Delegation** is the mechanism that makes this work. Control of the namespace is handed down in layers: from the root, to top-level domains, to individual organisations, and further into sub-zones. Each delegation boundary represents a change in ownership and authority. Importantly, the parent zone does not store or validate the child’s data — it only knows *where* to ask next. This separation of responsibility is what allows DNS to remain decentralised while still being globally navigable.

DNS also relies heavily on **query locality**. Most resolvers answer most questions from cache, often without contacting an authoritative server at all. This dramatically reduces global query load and latency, but it also means that different resolvers may temporarily hold different “views” of the same name. As a result, DNS prioritises availability and performance over immediate global agreement. Strong consistency would require constant coordination between authoritative servers and resolvers worldwide — something that would be slower, more fragile, and fundamentally unscalable at internet scale.

In distributed-systems terms, DNS is eventually consistent, not strongly consistent.

Because of this design, **propagation delays are not failures**. They are a natural consequence of delegation, caching, and independence between systems. DNS changes become visible as caches expire and queries refresh, not because the system is slow, but because it is deliberately cautious. DNS trades immediacy for resilience — and that trade is the reason it continues to function reliably across a network as large and diverse as the internet.

---

## 2. Namespaces, Zones, and Delegation

DNS is often explained as “names to IPs”, but the more accurate mental model is that DNS is a **global namespace** split into **administrative zones**, with control handed off through **delegation**. Understanding the boundaries between these concepts is what makes the rest of DNS make sense.

### Namespace
A **namespace** is the full structured space of DNS names.

It is:
- **hierarchical** (labels separated by dots)
- **rooted** at a single top (the root)
- **globally unique** by design (a given fully-qualified name has one intended meaning at a time)

Examples of names within the namespace:
- `github.com.`
- `unifi.home.`
- `_ldap._tcp.dc._msdcs.example.com.`

The dot at the end (`.`) is the **root label**. Most tools and humans omit it, but DNS itself treats fully-qualified names as rooted at `.`.

### Zone
A **zone** is an *administrative slice* of the namespace that is managed as a unit.

A zone contains:
- a set of DNS records for names within that portion of the namespace
- a **Start of Authority (SOA)** record identifying the zone and its control metadata
- one or more **NS** records that state which name servers are authoritative for the zone

A zone is not “a domain name”. A domain name is a *node in the namespace*. A zone is the portion of the namespace for which a particular set of authoritative servers publishes answers.

**Example**

Consider the namespace `example.com.`

If all records are managed together, then there is a single zone:

- Zone: `example.com.`
- Records served by this zone:
  - `example.com.`
  - `www.example.com.`
  - `mail.example.com.`
  - `vpn.example.com.`

All of these names live under the same zone and are answered by the same authoritative name servers.

---

### Zone Cut
A **zone cut** is the point in the namespace where responsibility is split: the parent zone stops being authoritative for a child subtree and instead delegates it.

**Example**

Starting from the previous zone:

- Parent zone: `example.com.`
- Child zone: `corp.example.com.`

If `example.com.` delegates `corp.example.com.` to different name servers, a zone cut occurs at `corp.example.com.`.

After the zone cut:
- `example.com.` remains authoritative for:
  - `example.com.`
  - `www.example.com.`
  - `mail.example.com.`
- `example.com.` is **no longer authoritative** for:
  - `corp.example.com.`
  - `host1.corp.example.com.`

Instead, `example.com.` publishes NS records that point to the authoritative servers for `corp.example.com.`, and control of that subtree moves entirely to the child zone.

The zone cut marks a **change in authority**, not just a naming boundary.

Zone cuts are implemented using:
- **NS records** in the parent zone (declaring the child zone’s authoritative servers)
- often **glue records** when needed (explained below)

## **Namespace vs Zones (Visualised)**

The DNS namespace is the full tree of names:

```
.
└── com.
    └── example.com.
        ├── www.example.com.
        ├── mail.example.com.
        └── corp.example.com.
            ├── host1.corp.example.com.
            └── host2.corp.example.com.

Zones are administrative boundaries applied *onto* that tree:

Zone: .
└── [delegates] com.
    Zone: com.
    └── [delegates] example.com.
        Zone: example.com.
        ├── www.example.com.
        ├── mail.example.com.
        └── [delegates] corp.example.com.   ← zone cut
            Zone: corp.example.com.
            ├── host1.corp.example.com.
            └── host2.corp.example.com.

```

The namespace never changes.
Zones are overlaid to define **where authority starts and stops**.

### Delegation vs Authority
These two words are easy to mix up:

- **Delegation** is what a **parent zone** does: it publishes NS (and sometimes glue) to say “for this child zone, ask these servers”.
- **Authority** is what the **child zone’s authoritative servers** have: they publish the actual data for names inside that zone.

The parent does not become authoritative for the child merely by listing its NS records. Delegation is a signpost; authority is the data source.

---

## How Zones Are Separated (Control Boundaries)

Zones are separated specifically to create **control boundaries**:
- different organisations can operate different parts of the namespace
- changes in one zone do not require coordination with unrelated zones
- operational responsibility can be delegated without transferring unrelated data

This is why the parent zone does not “own” child data.

The parent zone typically knows only:
- which name servers are authoritative for the child zone (NS records)
- sometimes the IP addresses of those name servers (glue, if required)

It does **not** store the child’s A/AAAA/MX/TXT/etc records, and it does not validate or curate them. The child zone is responsible for its own content.

---

## Why Glue Records Exist

A delegation points to name servers by **name**, not by IP. For example:

- `example.com.` delegates to `ns1.example.com.`

But this creates a potential bootstrapping problem:
- to find the IP of `ns1.example.com.`, you would need to query DNS
- but to query DNS for `example.com.`, you need to reach `ns1.example.com.`

That circular dependency is resolved using **glue records**.

**Glue records** are A/AAAA records published in the **parent zone** for the child’s name servers *when those name servers’ names are inside the child zone being delegated*.
They are copied into the parent zone for reachability, but remain authoritative only in the child zone.

They are not “extra convenience data”.
They are a practical necessity to prevent resolution loops and allow delegation to work reliably.

Glue is tightly scoped:
- it exists only to make the delegation reachable
- it is not a substitute for authoritative data in the child zone
- it can become stale if not managed carefully, which is one source of “DNS weirdness” at the edges

**Example**

Parent zone: com.  
Child zone: example.com.  
Name server for example.com.: ns1.example.com.

Delegation published by the parent:

example.com.    NS    ns1.example.com.

Problem:
- ns1.example.com. exists inside example.com.
- resolving it requires querying example.com.
- example.com. can only be reached via ns1.example.com.

Circular dependency.

Solution (glue):

example.com.         NS    ns1.example.com.
ns1.example.com.     A     192.0.2.53   ; glue

The glue record gives resolvers the IP address of the child zone’s name server so delegation can complete.

A common homelab setup is registering a domain with Cloudflare and then running your own authoritative DNS for part of it at home.

---

## The DNS Hierarchy (Root → TLD → Zone)

DNS hierarchy is built around a small number of structural layers:

### Root Zone (`.`)
The root zone is the top of the DNS namespace.
It does not contain “every record on the internet”.
It contains delegations to top-level domains.

Operationally, the root zone publishes:
- NS records for TLDs (e.g., `com.`, `org.`, `uk.`)
- the glue needed to reach those TLD name servers

### Top-Level Domains (TLDs)
TLD zones (like `com.` or `uk.`) publish delegations to second-level domains.

For example:
- `com.` delegates `example.com.`
- `uk.` delegates many second-level structures depending on registry policy (e.g., `co.uk.`)

A key point: the structure is always DNS hierarchy, but policy decisions can introduce additional levels (like `co.uk.`) before an organisation receives a delegation boundary.

### Authoritative Name Servers
**Authoritative name servers** publish the actual zone data for a zone and answer with authority for that zone.

They are authoritative because:
- the zone’s NS records name them as such
- they host the zone’s SOA and record sets

They do not need to be special hardware or “internet scale”.
In a homelab, your authoritative server might be:
- Unbound serving local zones
- AD DNS serving AD-integrated zones
- a BIND/PowerDNS instance serving internal zones

Authority is about publishing the zone, not about being globally famous.

---

## Delegation (Step-by-Step, Control Boundaries)

This is the key sequence, expressed as control boundaries rather than query mechanics:

1. **The root zone controls delegations to TLDs**  
   The root publishes which name servers are authoritative for each TLD zone.

2. **A TLD zone controls delegations to registered domains beneath it**  
   The `com.` zone publishes delegations for domains like `example.com.` by listing the authoritative name servers for that domain.

3. **The delegated domain becomes its own authority boundary**  
   The `example.com.` zone’s authoritative servers publish the records for names inside `example.com.`.

4. **Further sub-zones can be created by repeating the pattern**  
   `example.com.` may delegate `corp.example.com.` to different authoritative servers, creating a new zone cut and a new administrative boundary.

At each step:
- the parent publishes *where to go next*
- the child publishes *the data you actually want*

That separation of responsibilities is what makes DNS decentralised, scalable, and operationally survivable.

---

## 3. Recursive vs Iterative Resolution

DNS resolution is often described as “the client asks the server”, but that hides the real structure. In practice, DNS resolution is a **division of labour** between different actors, each with clearly limited responsibilities. Understanding this division explains why recursion exists, why most clients never perform it, and why it is carefully controlled.

---

### Stub Resolvers

A **stub resolver** is the DNS component built into an operating system or application.

Its characteristics:
- minimal logic
- no global knowledge of the DNS hierarchy
- no authority over any zone
- no responsibility for caching beyond very short-term local use

A stub resolver’s job is simple:
> “Ask a configured resolver for the answer and trust the response.”

Stub resolvers do **not**:
- walk the DNS hierarchy
- contact root or TLD servers
- make delegation decisions

This is why most end systems appear to “just ask DNS” and get an answer.

**Example: A Stub Resolver in the Real World**

When you run this command on a Linux system:

ping github.com

The DNS lookup is performed by the system’s **stub resolver**.

On modern Linux, that stub resolver is part of:
- the C library (`glibc`)
- often mediated by `systemd-resolved`

What actually happens:
- the application (`ping`) asks the OS: “What is the IP for github.com?”
- the stub resolver reads its configured DNS server (for example, `10.10.1.1`)
- it sends a single DNS query to that resolver
- it waits for a complete answer

It simply trusts the recursive resolver it has been configured to use for correctness and completeness.

In other words:
- your laptop
- your server
- your container
- your phone

…are all running stub resolvers.

---

### Recursive Resolvers

A **recursive resolver** is responsible for *doing the work* on behalf of stub resolvers.

When a stub resolver sends a query with recursion enabled, it is asking:
> “Find the final answer for me, no matter how many steps it takes.”

A recursive resolver:
- accepts full responsibility for resolving the name
- follows delegations from root → TLD → authoritative servers
- caches results to improve performance and reduce load
- returns a final answer (or a definitive failure)

Examples of recursive resolvers:
- Unbound
- Pi-hole
- ISP DNS resolvers
- Public resolvers (1.1.1.1, 8.8.8.8)

Recursive resolvers act as **trusted intermediaries** between simple clients and the global DNS system.

**Example: Unbound as a Trusted Intermediary (OPNsense)**

Assume:
- your laptop’s DNS server is `10.10.1.1`
- `10.10.1.1` is OPNsense running Unbound
- you open a browser and visit: https://github.com

Step 1: Stub resolver delegates trust  
The browser asks the OS:
“What is the IP address for github.com?”

The OS stub resolver:
- does not perform DNS resolution itself
- sends a single recursive query to `10.10.1.1`
- sets the RD (Recursion Desired) flag
- waits for a complete answer

At this point, the stub resolver’s job is finished.
It trusts Unbound to handle everything else.

---

Step 2: Unbound accepts responsibility  
Unbound receives the query and agrees to resolve it recursively.

Unbound now:
- takes full responsibility for resolution
- performs iterative queries to root, TLD, and authoritative servers
- validates responses
- applies caching rules and TTLs
- enforces policy (DNSSEC, local overrides, blocklists, etc.)

This work is intentionally hidden from the client.

---

Step 3: Unbound performs iterative resolution  
Unbound:
- queries a root server for `.`
- follows the delegation to `.com`
- follows the delegation to `github.com`
- receives the authoritative answer

Throughout this process:
- no authoritative server performs recursion
- each server answers only for what it controls
- Unbound decides where to query next

---

Step 4: Answer returned and cached  
Unbound returns the final IP address to the stub resolver.

Additionally:
- the result is cached according to its TTL
- future clients may be answered without external queries
- load on the global DNS infrastructure is reduced

---

Why Unbound is a “trusted intermediary”

From the client’s perspective:
- Unbound is trusted to recurse safely
- Unbound is trusted to cache correctly
- Unbound is trusted to enforce boundaries

From the internet’s perspective:
- Unbound does not expose recursion publicly
- Unbound never asks authoritative servers to recurse
- Unbound absorbs complexity on behalf of simple clients

This trust boundary is exactly what allows:
- simple clients
- protected authoritative servers
- and scalable DNS resolution
to coexist.

---

### Iterative Queries

An **iterative query** is a DNS query where the server responds with the *best information it has*, not a final answer.

In an iterative exchange:
- the server does **not** chase the answer
- it responds with referrals (delegations) when appropriate
- the responsibility for the next step lies with the requester

Root servers, TLD servers, and authoritative servers **only answer iteratively**. They never perform recursion on behalf of others.

This separation is deliberate:
- it prevents authority servers from becoming overloaded
- it preserves clear control boundaries
- it keeps the DNS hierarchy scalable

**Example: An Iterative Response (Web Browsing Context)**

Assume you type the following into a browser:

www.example.com

```
.        (root)
└── com.     (TLD)
    └── example.com.
        └── www.example.com.
```

Your computer’s **stub resolver** sends the request to a **recursive resolver**
(for example, your home DNS server or ISP resolver).

The recursive resolver now has to find the answer.

---

**Step 1: Asking the root**

The recursive resolver sends an **iterative query** to a root name server.

The root server does **not** return an IP address.
Instead, it replies with the best information it has:

- “I am not authoritative for www.example.com.”
- “Here are the name servers responsible for the .com zone.”

This reply is a **referral**, not an answer.

---

**Step 2: Asking the TLD**

The recursive resolver sends an iterative query to a `.com` name server.

The `.com` server again does **not** return an IP address.
It replies with:

- “I am not authoritative for www.example.com.”
- “Here are the name servers responsible for example.com.”

Again, this is a referral.

---

**Step 3: Asking the authority**

The recursive resolver now sends a query to an authoritative name server for
`example.com`.

This time, the server *is* authoritative and returns:

- “www.example.com resolves to 93.184.216.34.”

The recursive resolver caches the result and returns the IP address to your computer.
Your browser can now connect to the website.

---

At every step:
- each server answers only for the part of the namespace it controls
- no server performs recursion on behalf of the requester
- the recursive resolver decides where to query next

---

## What “Recursion” Actually Means

Recursion in DNS does **not** mean “a different kind of query”.

It means **delegated responsibility**.

When recursion is requested:
- the client delegates the entire resolution process to the resolver
- the resolver agrees to act on the client’s behalf
- the resolver absorbs complexity, latency, and failure handling

The recursive resolver becomes accountable for the result.

Without recursion, DNS would require every client to understand:
- the root servers
- TLD structure
- delegation chains
- caching strategy

That complexity is intentionally centralised instead.

---

## Why Most Clients Never Perform Iterative Resolution

Most clients:
- lack the logic to follow delegations
- lack the context to cache safely
- should not generate large volumes of DNS traffic
- should not depend on the availability of global infrastructure

For these reasons:
- stub resolvers do not talk to root or TLD servers
- iterative resolution is confined to recursive resolvers
- recursion concentrates complexity in a small number of well-managed systems

This design dramatically reduces global DNS load and keeps client systems simple and predictable.

---

## Where Recursion Is Allowed — and Forbidden

Recursion is **not universally permitted**.

- Recursive resolvers accept recursive queries from **trusted clients**
- Authoritative servers **do not offer recursion**
- Open recursion is considered dangerous and is widely blocked

This creates a **trust boundary**:
- clients trust recursive resolvers
- recursive resolvers trust authoritative answers
- authoritative servers trust no one to recurse on their behalf

Allowing recursion only within controlled boundaries prevents abuse, amplification attacks, and accidental overload.

## **Real-World Incidents: Why Recursion Is Restricted**

**Incident 1: The Spamhaus DNS Amplification Attack (2013)**

What went wrong:
Spamhaus, an anti-spam organisation, was targeted by a large DDoS attack.
Attackers abused thousands of **open recursive DNS resolvers** exposed to the internet.

Why it worked:
- The resolvers allowed recursion from anywhere.
- Attackers sent small DNS queries with a spoofed source IP (Spamhaus).
- Each resolver replied with a much larger response to the spoofed address.
- Traffic was amplified many times over.

Impact:
- Traffic volumes reached hundreds of gigabits per second.
- Not only Spamhaus, but parts of the surrounding internet infrastructure were affected.
- The DNS servers themselves were victims as well as participants.

What changed:
- DNS operators aggressively closed open resolvers.
- ISPs began blocking recursion from untrusted networks by default.
- Modern DNS software now ships with recursion disabled on public-facing servers.

This incident cemented the rule:
authoritative servers must never recurse, and recursive resolvers must never be open.

---

**Incident 2: Authoritative Servers Accidentally Offering Recursion**

What went wrong:
In multiple documented outages (enterprise and ISP environments),
authoritative DNS servers were misconfigured to allow recursion.

Why it mattered:
- These servers were designed to answer only for their own zones.
- Once recursion was enabled, they began performing lookups for unrelated domains.
- Each external lookup consumed CPU, memory, and outbound bandwidth.

Impact:
- Legitimate authoritative queries were delayed or dropped.
- DNS latency increased dramatically.
- In some cases, the authoritative zones appeared “down” despite the service running.

What changed:
- Clear separation of roles became standard practice:
  - authoritative-only servers
  - recursive-only resolvers
- Configuration defaults were hardened to prevent role mixing.
- Operational guidance shifted from “it works” to “it must be isolated”.

This reinforced the idea that recursion is not just a feature —
it fundamentally changes the load profile and risk of a DNS server.

---

**Incident 3: ISP Open Resolvers Used as Infrastructure Weapons**

What went wrong:
Several ISPs historically deployed recursive resolvers
intended only for customers — but exposed them to the public internet.

Why it was dangerous:
- Attackers discovered these resolvers via scanning.
- The resolvers were then used repeatedly in amplification attacks.
- The ISPs’ own networks suffered collateral damage.

Impact:
- ISP address ranges were rate-limited or blackholed by upstream providers.
- Legitimate customers experienced DNS failures.
- Trust in the ISP’s network reliability was damaged.

What changed:
- Recursion was restricted to internal address ranges.
- Source address validation and filtering became more common.
- Recursive resolvers were moved behind clear network boundaries.

This is why modern ISP resolvers:
- accept recursion only from customer IP ranges
- refuse queries from the wider internet entirely.

---

### **Design Constraint: Why Root and TLD Servers Never Recurse**

This is not hypothetical — it is a preventative design choice.

If root or TLD servers offered recursion:
- every client could offload full resolution work onto them
- global query volume would explode
- delegation boundaries would collapse

Instead:
- root and TLD servers answer **only iteratively**
- they return referrals and stop
- they are protected from abuse by design, not configuration

This separation is one of the reasons DNS has remained stable
despite decades of traffic growth and repeated attacks.

---

## RD and RA Flags

DNS uses two flags to signal recursion behaviour:

- **RD (Recursion Desired)**  
  Set by the client to request recursion.

- **RA (Recursion Available)**  
  Set by the server to indicate it supports recursion.

Typical patterns:
- Stub → Recursive resolver: RD set, RA returned
- Recursive → Authoritative server: RD unset, RA unset
- Authoritative server → anyone: RA never set

These flags do not force behaviour — they communicate **intent and capability**. A server may ignore RD, and a client must not assume recursion unless RA is set.

---

## Resolution as a Division of Labour

DNS resolution works because each participant does *only one job*:

- Stub resolvers ask questions
- Recursive resolvers do the hard work
- Authoritative servers publish data
- Root and TLD servers provide structure, not answers

Recursion is not a convenience feature.
It is the mechanism that allows DNS to scale while keeping clients simple and authority boundaries intact.

---

## 4. DNS Transport Mechanics

DNS looks simple at the application layer, but its transport choices are deliberate and tightly constrained by history, performance requirements, and scale. DNS is not built on a single transport for all cases; instead, it uses **UDP by default**, with **TCP as a fallback and for specific mandatory scenarios**. Understanding why explains many real-world DNS failure modes.

---

## UDP vs TCP in DNS

### UDP (Default)

DNS primarily uses **UDP** for queries and responses.

Why UDP is the default:
- minimal overhead (no connection setup)
- low latency
- well-suited to short request/response exchanges
- scales efficiently at global query volumes

Most DNS lookups are:
- small
- cacheable
- latency-sensitive

UDP allows resolvers to issue queries and receive answers quickly without maintaining state between packets. This makes it ideal for the vast majority of DNS traffic.

---

### TCP (Fallback and Mandatory Use)

DNS uses **TCP** when:
- a response does not fit in a single UDP packet
- zone transfers are performed (AXFR / IXFR)
- DNSSEC responses exceed UDP size limits
- reliability is explicitly required

TCP provides:
- guaranteed delivery
- ordered packets
- congestion control

But it also introduces:
- connection setup overhead
- higher latency
- increased server resource usage

For this reason, TCP is used **only when necessary**, not by default.

---

## Port 53

DNS uses **port 53** for both UDP and TCP.

This consistency:
- simplifies firewall rules
- allows easy fallback from UDP to TCP
- avoids ambiguity in transport handling

Blocking TCP/53 while allowing UDP/53 is a common misconfiguration and causes subtle failures, especially with DNSSEC-enabled domains.

---

## Message Size Limits

### Traditional DNS Size Limits

Historically:
- DNS over UDP was limited to **512 bytes**
- responses larger than this could not be delivered via UDP

This constraint shaped DNS design:
- compact record formats
- minimal responses
- referral-heavy resolution

---

### EDNS(0) and Modern Limits

Modern DNS uses **EDNS(0)** to advertise larger acceptable UDP payload sizes (commonly ~1232–4096 bytes).

EDNS(0):
- does not change DNS itself
- allows clients and servers to agree on larger UDP messages
- reduces unnecessary TCP fallback

However:
- larger packets are more likely to be fragmented
- fragmented packets are more likely to be dropped

This creates a practical tension between efficiency and reliability.

---

## Truncation and Fallback

When a DNS response:
- exceeds the advertised UDP size
- cannot be safely delivered in a single UDP packet

The server sets the **TC (Truncated)** flag in its response.

What happens next:
- the resolver discards the truncated response
- the resolver retries the query over **TCP**
- the full response is delivered reliably

This mechanism is intentional:
- UDP is attempted first for speed
- TCP is used only when required

Resolvers are expected to handle this automatically.

---

## Why UDP Was Chosen Historically

DNS was designed in an era where:
- networks were slower
- connections were expensive
- stateful protocols were avoided where possible

UDP allowed DNS to be:
- stateless on the server side
- fast to process
- resilient to individual query loss

Rather than relying on transport-level reliability, DNS:
- tolerates dropped queries
- relies on retries
- uses caching to reduce repeated lookups

Reliability is handled **above the transport layer**, not by the transport itself.

---

## Why DNS Is Sensitive to Packet Loss and MTU Issues

MTU = Maximum Transmission Unit

In simple terms, it’s the largest packet size that can be sent over a network link without being split up.

Because DNS relies heavily on UDP:
- lost packets are not retransmitted automatically
- fragmented packets may never reassemble
- path MTU issues can silently drop responses

Common failure patterns include:
- DNSSEC-enabled domains failing to resolve
- large TXT or DNSSEC responses timing out
- intermittent resolution failures depending on path

From the resolver’s perspective:
- a lost response looks identical to a slow or unreachable server
- retries may succeed or fail unpredictably

This is why:
- DNS issues often appear “flaky”
- MTU misconfigurations cause disproportionate pain
- blocking TCP/53 breaks resolution in non-obvious ways

---

## Transport Design as a Trade-off

DNS transport choices reflect a consistent philosophy:
- optimise for the common case
- accept occasional retries
- push complexity to resolvers, not authorities

UDP provides speed and scale.
TCP provides correctness when needed.
Together, they allow DNS to function efficiently across unreliable and heterogeneous networks.

Transport is not an implementation detail in DNS.
It is a core part of how DNS achieves global scale.

## 5. DNS Record Types (In Depth)

DNS record types define **what kind of information a name represents** and, just as importantly, **how that information is allowed to behave**. 
Each record type carries specific semantics and constraints that shape how clients, resolvers, and applications interpret DNS data.

This section focuses on *meaning and limitations*, not configuration syntax.

---

## 5.1 Address Records

### A (IPv4)
An **A record** maps a name to an IPv4 address.

Semantics:
- represents a concrete network endpoint
- implies direct reachability over IPv4
- may appear multiple times for the same name (multi-homing)

**Example: A Record**

Name: web.example.com.

web.example.com.    A    203.0.113.10

Meaning:
- web.example.com resolves to an IPv4 address
- clients can reach the service over IPv4
- the address represents a real network endpoint

If multiple A records exist:
- the name represents multiple IPv4 endpoints
- clients may choose between them for redundancy or load distribution

---

### AAAA (IPv6)
An **AAAA record** maps a name to an IPv6 address.

Semantics:
- represents the same logical endpoint as an A record
- uses a different address family
- may coexist with A records for the same name

**Example: AAAA Record**

Name: web.example.com.

web.example.com.    AAAA    2001:db8:abcd::10

Meaning:
- web.example.com resolves to an IPv6 address
- the service is reachable over IPv6
- the endpoint is logically the same service as the IPv4 one

If both A and AAAA exist:
- DNS advertises multiple address families
- DNS does not choose which one is used
- the client decides which address to try

---

### Why Both May Exist

A name may publish both A and AAAA records to:
- support both IPv4 and IPv6 clients
- allow gradual IPv6 adoption
- provide redundancy across address families

The presence of both does **not** imply preference or priority.
It merely advertises available options.

**Combined Example**

web.example.com.    A       203.0.113.10
web.example.com.    AAAA    2001:db8:abcd::10

This means:
- the service is reachable over both IPv4 and IPv6
- DNS makes both options visible
- client behaviour determines which address is actually used

---

### Client Selection Behaviour

DNS itself does not choose which address to use.

Instead:
- resolvers return all applicable records
- the client or application decides which address to try
- selection logic is OS- and application-dependent

This means:
- different systems may choose different addresses
- behaviour can vary between operating systems
- DNS cannot enforce address-family preference

---

### IPv6 Preference Pitfalls

When AAAA records exist:
- IPv6-capable clients may prefer IPv6
- applications may attempt IPv6 first even if connectivity is poor

Common pitfalls:
- broken or partially working IPv6 paths
- slower fallback to IPv4
- intermittent or hard-to-diagnose failures

Publishing AAAA records implicitly asserts:
> “IPv6 connectivity to this service is reliable.”

If that assertion is untrue, user experience suffers.

---

## 5.2 Alias and Indirection

### CNAME
A **CNAME (Canonical Name)** record creates an alias from one name to another.

Semantics:
- the alias name has *no data of its own*
- all resolution continues at the target name
- the alias and target are treated as the same entity

Constraints:
- a name with a CNAME **cannot have any other records**
- this includes A, AAAA, MX, TXT, etc.

This restriction exists to prevent ambiguity:
a name cannot be both “this” *and* “that”.

**Example: CNAME (Alias)**

Alias name: app.example.com.  
Target name: web.example.net.

app.example.com.    CNAME    web.example.net.

Meaning:
- app.example.com has no records of its own
- all resolution continues at web.example.net
- clients treat both names as the same endpoint

Important constraint:
- app.example.com cannot have A, AAAA, MX, TXT, or any other records
- all data must exist on web.example.net

This prevents ambiguity:
app.example.com is either an alias, or a name with its own data — never both.

---

### DNAME
A **DNAME** record provides redirection for an entire subtree.

Semantics:
- rewrites all names beneath a node to a different subtree
- operates at the namespace level, not individual names

DNAME is:
- powerful
- rarely used
- poorly understood by many applications

For this reason, DNAME is uncommon outside specialised environments.

**Example: DNAME (Subtree Redirection)**

Zone contains the following DNAME:

legacy.example.com.    DNAME    new.example.net.

Meaning:
- any name under legacy.example.com is rewritten
- resolution continues under new.example.net

Examples:
- host1.legacy.example.com → host1.new.example.net
- api.v1.legacy.example.com → api.v1.new.example.net

What DNAME does:
- redirects an entire subtree at once
- applies automatically to all descendants
- does not require individual aliases

What DNAME does not do:
- it does not alias legacy.example.com itself
- it does not copy records
- it does not guarantee application support

---

### Why CNAME Chains Are Discouraged

A **CNAME chain** occurs when:
- one CNAME points to another CNAME
- resolution requires multiple indirections

Problems caused by chains:
- increased lookup latency
- higher risk of failure
- greater dependence on multiple zones and operators

While allowed by DNS:
- long chains are fragile
- operationally discouraged
- difficult to debug when broken

DNS indirection should be shallow and intentional.

**Example: A CNAME Chain (What Not to Do)**

service.example.com.    CNAME    frontend.example.net.
frontend.example.net.   CNAME    lb.provider.com.
lb.provider.com.        CNAME    node123.region.provider.net.
node123.region.provider.net.    A    203.0.113.45

What this means:
- resolving service.example.com requires following four names
- each step may involve a different DNS zone
- each zone may be operated by a different party

Why this is a problem:
- every additional lookup adds latency
- failure at any link breaks the entire chain
- troubleshooting requires visibility into multiple zones
- caching behaviour becomes harder to predict

This works — until it doesn’t.

One level of indirection is reasonable. More than that is technical debt.

---

## 5.3 Mail and Text Records

### MX (Mail Exchange)
An **MX record** identifies servers responsible for receiving mail for a domain.

Semantics:
- mail delivery is separated from general name resolution
- multiple servers can be listed for redundancy

---

### Priority Handling (MX)

MX records include a **priority value**:
- lower numbers are preferred
- higher numbers act as fallback

Mail delivery logic:
- attempt lowest-priority server first
- try higher-priority servers only if delivery fails

DNS itself does not load-balance mail.
Priority expresses *order*, not distribution.

**Example: MX Records (Mail Routing, Not Hosting)**

Domain: example.com.

example.com.    MX    10 mail-primary.example.net.
example.com.    MX    20 mail-backup.example.net.

Meaning:
- mail for @example.com is delivered to mail servers, not to example.com itself
- mail-primary.example.net is tried first (priority 10)
- mail-backup.example.net is used only if the primary fails

Important:
- DNS does not distribute mail across both servers
- priority defines order, not load balancing
- delivery attempts are controlled by the sending mail system, not DNS

**Example: MX Priority in Failure Scenarios**

example.com.    MX    10 mail1.example.net.
example.com.    MX    20 mail2.example.net.

If mail1.example.net:
- is unreachable
- refuses the connection
- times out

Then:
- the sending system retries using mail2.example.net

If mail1 later recovers:
- it resumes receiving mail
- mail2 returns to standby

MX priorities describe preference, not traffic share.

---

### TXT (Text)
A **TXT record** stores arbitrary text associated with a name.

Originally intended for:
- human-readable annotations
- free-form metadata

In practice, TXT has become a **general-purpose signalling mechanism**.

**Example: TXT Records (Protocol Signalling)**

example.com.    TXT    "v=spf1 ip4:203.0.113.0/24 -all"

Meaning:
- this TXT record is not for humans
- it is interpreted by mail systems as an SPF policy
- it declares which systems are allowed to send mail for example.com

TXT records carry structured data:
- DNS stores the text
- applications interpret the meaning
- DNS itself enforces nothing

**Example: TXT as a General Signalling Channel**

example.com.    TXT    "google-site-verification=abc123"
example.com.    TXT    "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"

Meaning:
- unrelated applications read different TXT records
- DNS does not validate or understand the contents
- TXT acts as a generic data publication mechanism

This flexibility is why TXT became a protocol escape hatch: it allows new systems to publish data without changing DNS itself.

---

### SPF, DKIM, and DMARC (High-Level, With Context)

Modern email authentication exists to answer three questions:

1. **Is this system allowed to send mail for this domain?** (SPF)
2. **Was this message altered after it was sent?** (DKIM)
3. **What should receivers do if authentication fails?** (DMARC)

All three are published via **TXT records**, but they serve distinct roles.

---

## SPF — Sender Policy Framework

**What SPF is**

SPF allows a domain to declare **which systems are permitted to send email on its behalf**.

It answers:
> “Is this sending IP authorised to send mail claiming to be from this domain?”

SPF is checked against:
- the **sending server’s IP address**
- the **domain in the message’s envelope sender**

---

**SPF Example**

example.com.    TXT    "v=spf1 ip4:203.0.113.0/24 include:mail.provider.net -all"

Meaning:
- mail may be sent from:
  - any IP in 203.0.113.0/24
  - servers authorised by mail.provider.net
- mail from any other source should fail (`-all`)

What SPF does:
- validates *where* the mail came from

What SPF does **not** do:
- validate message content
- protect against message modification
- enforce rejection by itself

SPF is advisory — receivers decide how to act on failures.

---

## DKIM — DomainKeys Identified Mail

**What DKIM is**

DKIM provides **cryptographic verification** that:
- a message was authorised by the domain
- specific parts of the message were not altered in transit

It answers:
> “Was this message signed by the domain, and has it been tampered with?”

DKIM works by:
- signing outgoing mail with a private key
- publishing the matching public key in DNS

---

**DKIM Example**

selector1._domainkey.example.com.    TXT    "v=DKIM1; k=rsa; p=MIIBIjANBgkq..."

Meaning:
- example.com publishes a public key
- receiving mail servers use it to verify DKIM signatures
- a valid signature proves domain-level authorisation and integrity

What DKIM does:
- protects message headers and body
- survives forwarding better than SPF

What DKIM does **not** do:
- say anything about sending IP authorisation
- tell receivers what to do on failure

---

## DMARC — Domain-based Message Authentication, Reporting, and Conformance

**What DMARC is**

DMARC ties SPF and DKIM together and adds **policy**.

It answers:
> “If authentication fails, what should the receiver do?”

DMARC also defines:
- reporting
- alignment rules
- enforcement expectations

---

**DMARC Example**

_dmarc.example.com.    TXT    "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"

Meaning:
- messages claiming to be from example.com must pass SPF or DKIM
- failures should be rejected
- reports should be sent to dmarc@example.com

DMARC introduces **alignment**:
- SPF and/or DKIM must authenticate *the same domain* visible to users
- prevents lookalike and spoofing attacks

---

## How They Work Together (One Flow)

1. A mail server receives a message claiming to be from `user@example.com`
2. The receiver:
   - checks SPF → is the sending IP authorised?
   - checks DKIM → is the message signed and intact?
3. DMARC evaluates the results:
   - did SPF or DKIM pass?
   - are they aligned with example.com?
4. DMARC policy determines the outcome:
   - none → monitor only
   - quarantine → treat as suspicious
   - reject → refuse the message

No single mechanism is sufficient alone.
They are designed to work **as a system**.

---

## Why TXT Is Used (and Why It’s a Problem)

TXT was chosen because:
- DNS already existed everywhere
- TXT allowed arbitrary data
- no DNS protocol changes were required

This made TXT a **protocol escape hatch**:
- fast to deploy
- universally readable
- easy to extend

But it also means:
- TXT records now carry critical security policy
- unrelated systems overload the same mechanism
- DNS data becomes semantically dense and opaque

DNS publishes the data.
Mail systems interpret and enforce it.

DNS itself remains neutral.

---

## 5.4 Service Discovery

### SRV (Service)
An **SRV record** describes:
- the location of a service
- the port it runs on
- relative priority and weight

Semantics:
- decouples service identity from hostnames
- allows services to move without changing clients
- supports prioritisation and basic load distribution

SRV records were designed to solve real service-discovery problems.

**Example: SRV Record (Service Discovery)**

Service: LDAP for example.com.

_ldap._tcp.example.com.    SRV    10 60 389 ldap1.example.com.
_ldap._tcp.example.com.    SRV    20 20 389 ldap2.example.com.

Meaning:
- the LDAP service for example.com is provided by multiple servers
- ldap1.example.com is preferred (priority 10)
- ldap2.example.com is a fallback (priority 20)
- weight (60 / 30) controls selection between servers of the same priority
- clients learn both the hostname and port from DNS

What this enables:
- clients do not hard-code server names
- services can move between hosts
- ports can change without client reconfiguration

DNS publishes service location.

---

### URI Records (Briefly)

**URI records** extend service discovery by:
- publishing full URIs
- including protocol and scheme information

They are:
- standards-based
- sparsely implemented
- rarely relied upon in practice

---

### Why Many Applications Ignore SRV

Despite their usefulness, many applications:
- assume fixed ports
- hard-code hostnames
- perform their own discovery logic

Reasons include:
- historical precedent
- simplicity
- inconsistent client support

As a result:
- SRV is widely used in environments like Active Directory
- but often ignored by general-purpose applications

DNS provides the mechanism. Applications decide whether to respect it.

---

### Key Takeaway

DNS record types encode **intent**, not behaviour.

They describe:
- what a name represents
- how it may be used
- what constraints apply

Correct DNS design depends less on syntax and more on respecting those semantics.

## 6. Time, TTLs, and Caching Semantics

DNS works because it remembers answers.  
Caching is not an optimisation layered on top of DNS — it is a **core design contract**. Understanding that contract explains why DNS behaves predictably even when it feels misleading.

---

## TTL — Time To Live

A **TTL** defines how long a DNS answer may be cached before it must be discarded.

Semantics:
- TTL is expressed in seconds
- it is attached to individual records
- it is authoritative metadata, not a client suggestion

When a resolver receives a record with a TTL:
- it may reuse that answer until the TTL expires
- it must not reuse it after expiry
- it does not need to revalidate early

TTL answers one question only:
> “How long is this answer allowed to be reused?”

---

## TTL Is a Contract, Not a Hint

Once a resolver caches a record:
- it is **allowed** to reuse it
- it is **not required** to re-check it early
- it is **not obligated** to notice upstream changes

This is why DNS appears to “lie”:
- resolvers are honouring the contract
- not behaving incorrectly

Lowering a TTL affects **future caches only**.
It does nothing to invalidate answers already cached elsewhere.

---

## Minimum TTLs

Resolvers may impose **minimum TTLs**, ignoring very low values.

Reasons include:
- protection against excessive query volume
- defence against accidental or malicious TTL abuse
- operational stability

This means:
- a published TTL of 5 seconds does not guarantee 5-second propagation
- different resolvers may enforce different minimums
- behaviour varies across implementations

TTL expresses permission, not enforcement.

---

## Negative Caching

DNS also caches **failures**.

When a resolver asks:
- “Does this name exist?”

And the authoritative answer is:
- “No” (NXDOMAIN)
- or “No data for this record type”

That negative result may be cached.

This is called **negative caching**.

---

## Negative TTLs and SOA

Negative caching duration is controlled by the zone’s **SOA record**, specifically:
- the **negative TTL** (often the last field)

Semantics:
- a non-existent name may be cached as non-existent
- adding the record later does not take effect immediately
- resolvers are allowed to continue answering “no” until expiry

This is a common source of confusion:
> “I added the record, but it still doesn’t resolve.”

The cache is not stale.
It is contractually correct.

---

## SOA Timers (Authoritative Synchronisation)

The **SOA record** also defines timers used by authoritative servers.

Key fields:
- **Refresh** — how often secondaries check for updates
- **Retry** — how long to wait before retrying after failure
- **Expire** — how long a secondary may serve data without contact
- **Negative TTL** — how long negative answers may be cached

These timers do **not** affect client-side caching directly.
They govern:
- zone transfer behaviour
- authoritative data freshness
- fail-safe operation between primaries and secondaries

Confusing SOA timers with record TTLs is a common mistake.

---

## Why “Lower the TTL” Is Not a Magic Fix

Lower TTLs:
- reduce future cache lifetime
- increase query volume
- increase resolver load

They do **not**:
- invalidate existing caches
- guarantee immediate propagation
- override negative caching already in place

In practice:
- TTL changes must be planned **in advance**
- emergency changes are limited by existing cache state
- DNS cannot be made instant without breaking its design

---

## How Caches Are Allowed to Lie

Resolvers are allowed to:
- return cached data until TTL expiry
- return cached NXDOMAIN responses
- ignore upstream changes during that window

From DNS’s perspective:
- this is correct behaviour
- this is what enables scale and resilience

DNS chooses:
- predictability over immediacy
- contracts over coordination
- eventual convergence over instant agreement

---

## Caching as a Design Choice

DNS caching is not about speed alone.

It:
- protects authoritative servers
- reduces global traffic
- absorbs transient failures
- allows DNS to function during partial outages

The cost is time-based inconsistency.

Once that trade-off is understood, DNS caching stops being surprising and starts being reliable (though sometimes an exercise in patience).

**Example: “It Works on My Phone but Not at My Friend’s House”**

You register a new domain:

superawesomehomelabdomain.com.

You add this record at 10:00:

superawesomehomelabdomain.com.    A    203.0.113.42    (TTL: 3600)

---

**What happens next**

At 10:01:
- your phone (on mobile data) looks up the domain
- it queries its ISP’s recursive resolver
- the resolver has no cached answer
- it asks the authoritative servers
- it receives the new record and caches it for 3600 seconds

Result:
- the site works on your phone

---

At 10:05:
- you visit a friend
- their home network uses a *different* recursive resolver
- that resolver previously queried the domain before you added the record
- it cached a negative result (NXDOMAIN)

That negative result:
- is still valid
- is allowed to be reused
- has not yet expired

Result:
- the domain does not resolve at your friend’s house

---

**Nothing is broken**

Both resolvers are behaving correctly:

- your mobile ISP resolver cached a *positive* answer
- your friend’s resolver cached a *negative* answer
- both are honouring their TTL contracts

DNS has not “half updated”.
Different caches simply have different valid views at the same time.

---

**When it fixes itself**

Eventually:
- the negative cache at your friend’s resolver expires
- it queries the authoritative servers again
- it learns the domain now exists
- it caches the new answer

At that point:
- the site works everywhere
- without any further changes

---

**Why this happens**

DNS propagation is not a push.
It is:
- cache expiry
- followed by re-query
- followed by gradual convergence

Lowering TTLs only affects:
- future cache entries
- not ones that already exist

---

**Key takeaway**

If a DNS change works in one place but not another:
- caches are out of sync
- not misconfigured

DNS is converging exactly as designed.

## 7. Authoritative DNS Behaviour

Authoritative DNS is not primarily about answering questions.  
It is about **publishing data** and ensuring that publication is **consistent, durable, and recoverable** across multiple servers.

Resolvers consume DNS.
Authoritative servers **publish it**.

Understanding authoritative DNS as a *publication system* explains how primaries, secondaries, transfers, and serial numbers fit together.

---

## Primary vs Secondary Authoritative Servers

An authoritative zone is typically served by multiple servers.

- **Primary (master)**  
  - holds the editable copy of the zone  
  - is where changes are made  
  - is the source of truth for that zone’s data  

- **Secondary (slave)**  
  - holds a replicated copy of the zone  
  - does not accept edits  
  - serves data on behalf of the zone  

From a resolver’s perspective:
- both are equally authoritative
- answers from either are valid

The distinction exists for **publication**, not resolution.

---

## Zone Transfers (AXFR and IXFR)

Secondaries stay in sync using **zone transfers**.

### AXFR — Full Transfer
An **AXFR** transfers the entire zone.

Characteristics:
- complete copy of all records
- higher bandwidth usage
- typically used for:
  - initial sync
  - recovery
  - small zones

---

### IXFR — Incremental Transfer
An **IXFR** transfers only changes since the last known version.

Characteristics:
- lower bandwidth
- faster updates
- preferred when supported by both servers

Both mechanisms exist because:
- DNS zones vary wildly in size
- reliability matters more than elegance

---

## Serial Numbers (Why They Matter)

Each zone has a **serial number**, stored in the SOA record.

The serial:
- represents the version of the zone
- increases when the zone changes
- allows secondaries to detect updates

The serial is not optional metadata.
It is the **core change-detection mechanism** for authoritative DNS.

If the serial does not increase:
- secondaries assume nothing changed
- no transfer occurs
- stale data persists indefinitely

---

## How Secondaries Stay in Sync

The basic publication loop looks like this:

1. The secondary checks the primary’s SOA record
2. It compares the serial number to its own
3. If the serial is higher:
   - it requests a transfer (IXFR or AXFR)
4. If the serial is unchanged:
   - it does nothing

This process is governed by SOA timers:
- **Refresh** — how often to check
- **Retry** — how long to wait after failure
- **Expire** — how long data may be served without contact

Secondaries do not “poll constantly”.
They follow the contract defined by the zone.

---

## NOTIFY (Speeding Up Publication)

Waiting for refresh intervals can be slow.

**NOTIFY** allows the primary to proactively inform secondaries:
> “The zone has changed. Check now.”

NOTIFY:
- does not transfer data itself
- simply triggers an immediate SOA check
- reduces propagation delay between authorities

NOTIFY improves *freshness*, not correctness.
The serial number still controls whether a transfer occurs.

---

## What Happens When Transfers Fail

When a secondary cannot contact the primary:

- it continues serving its existing data
- it retries according to the Retry timer
- it does not discard data immediately

If the **Expire** timer is reached:
- the secondary stops answering authoritatively
- the zone is considered invalid

This behaviour prioritises:
- availability over immediacy
- last-known-good data over silence

Serving slightly stale data is preferable to serving none.

---

## Failure Modes and Their Effects

Common authoritative failures include:

- **Serial not incremented**
  - secondaries never update
  - zone appears “stuck”

- **Transfers blocked**
  - secondaries fall behind
  - eventually expire

- **NOTIFY blocked**
  - updates still propagate
  - but only after refresh intervals

None of these affect DNS resolution immediately.
They affect **publication correctness over time**.

---

## Authoritative DNS as a Publication System

Authoritative DNS behaves like a distributed publishing pipeline:

- primaries publish new editions
- serial numbers identify versions
- secondaries subscribe and replicate
- resolvers consume whatever edition they reach

Queries are incidental.
Publication is the real job.

## 8. Active Directory–Integrated DNS

Active Directory does not merely *use* DNS — it is **structurally dependent** on it.  
Without DNS behaving correctly, Active Directory cannot locate itself, replicate, or function as a directory at all.

AD-integrated DNS should be understood as a **directory dependency**, not a Windows feature.

---

## Why Active Directory Requires DNS

Active Directory is a **distributed directory**.
Clients and domain controllers must constantly discover:

- domain controllers
- global catalog servers
- authentication services
- replication partners

Hard-coding this information would be brittle and unscalable.
DNS provides the discovery fabric AD depends on.

If DNS is wrong:
- authentication fails
- logons slow or break
- replication stalls
- the directory fragments

Active Directory assumes DNS is:
- authoritative
- consistent
- dynamically updateable

If those assumptions are violated, AD degrades rapidly.

---

## How Active Directory Uses SRV Records

AD relies heavily on **SRV records** to publish service locations.

Rather than asking:
> “What is the IP of a domain controller?”

AD-aware clients ask:
> “Where is the LDAP / Kerberos / GC service for this domain?”

These questions are answered via DNS.

---

### Example: Domain Controller Discovery

Domain: corp.example.com.

_corp._tcp.dc._msdcs.corp.example.com.    SRV    0 100 389 dc1.corp.example.com.
_corp._tcp.dc._msdcs.corp.example.com.    SRV    0 100 389 dc2.corp.example.com.

Meaning:
- multiple domain controllers exist
- all provide the same directory services
- clients can choose any suitable DC
- no DC is hard-coded

This allows:
- DCs to be added or removed
- services to move
- load to distribute naturally

DNS is the directory’s locator service.

---

## Secure Dynamic Updates

Active Directory requires DNS records to be:
- created automatically
- updated as systems change
- protected from unauthorised modification

This is achieved through **secure dynamic updates**.

Semantics:
- domain-joined systems can update their own records
- domain controllers update service records
- updates are authenticated and authorised
- arbitrary systems cannot overwrite records

This keeps DNS:
- accurate
- current
- resistant to spoofing

Manual DNS management does not scale for AD environments.

---

## AD Zones Stored in Active Directory

In AD-integrated DNS:
- DNS zones are stored **inside the directory**
- zone data replicates using AD replication
- no single DNS master exists

This changes the model entirely.

There is:
- no primary DNS server
- no AXFR/IXFR between DNS servers
- no single point of update

DNS becomes **multi-master**, just like AD itself.

---

## Multi-Master Implications

Because DNS data is stored in AD:

- any domain controller hosting DNS can accept updates
- changes replicate automatically via AD
- DNS consistency depends on AD health

This provides:
- high availability
- simplified management
- tight integration

But it also means:
- DNS failures often indicate directory replication issues
- DNS problems are rarely “just DNS” in AD environments

---

## Example: Multi-Master DNS in Practice

You update a DNS record on DC1.

What happens:
- the change is written to Active Directory
- AD replication distributes the update
- DC2 and DC3 learn the change automatically
- all DNS servers become consistent over time

There is no zone transfer.
There is no DNS serial number.
The directory handles versioning and replication.

DNS publication is now a directory concern.

---

## Common Failure Modes

Because of the tight coupling, common AD DNS failures include:

- **Broken SRV records**
  - clients cannot locate domain controllers
  - logons fail or slow dramatically

- **Replication issues**
  - DNS records differ between DCs
  - clients see inconsistent directory state

- **Dynamic update failures**
  - machines cannot register records
  - stale records accumulate
  - name resolution becomes misleading

- **Using non-AD-aware DNS**
  - SRV records missing or incorrect
  - secure updates unavailable
  - directory assumptions violated

These failures often present as:
> “Active Directory is broken”

When the root cause is DNS.

---

## Active Directory DNS as a Dependency

Active Directory treats DNS as:
- a discovery mechanism
- a publication system
- a replication substrate

Not as a convenience.

If DNS does not:
- allow secure updates
- publish SRV records correctly
- replicate consistently

Then Active Directory cannot function reliably.

Understanding AD DNS means understanding that **the directory is built on top of DNS, not alongside it**.

## 9. Split Horizon and Views

Split-horizon DNS is a **deliberate violation of the idea that DNS has one true answer**.

It exists because networks have different audiences, different trust boundaries, and different routing realities. DNS accommodates this by allowing the *same name* to resolve differently depending on **who is asking**.

This is intentional behaviour, not inconsistency.

---

## What Split-Horizon DNS Is

Split-horizon DNS means:
- the same DNS name
- returns different answers
- based on the source of the query

Typically:
- internal clients receive internal addresses
- external clients receive public addresses

The DNS name remains the same.
Only the *view* of the namespace changes.

---

## Views

A **view** is a policy-defined lens through which DNS answers are generated.

Views may be selected based on:
- source IP address
- network location
- interface or listener
- authentication context (in some systems)

Each view may have:
- its own zone data
- different answers for the same name
- different recursion and caching rules

Views allow DNS to present **multiple coherent realities** at the same time.

---

## Example: Internal vs External Answers

Domain: superawesomehomelabdomain.com.

Internal view answer:
nas.superawesomehomelabdomain.com.    A    10.10.1.50

External view answer:
nas.superawesomehomelabdomain.com.    A    203.0.113.50

Meaning:
- internal clients connect directly over the LAN
- external clients connect via NAT, reverse proxy, or VPN
- the hostname stays stable
- the network path changes

DNS is used to steer traffic appropriately.

---

## Why This Exists

Split-horizon DNS exists because:
- private IPs are not routable externally
- external access often requires firewalls or proxies
- hairpin routing is inefficient or impossible
- services expose different interfaces to different audiences

Without split horizon:
- internal clients may be forced out to the internet and back in
- external clients may receive unusable private addresses
- security boundaries become harder to enforce

Split horizon is about **reachability**, not deception.

---

## Why This Breaks Naive Troubleshooting

Split-horizon DNS breaks the assumption:
“DNS should return the same answer everywhere.”

Common failure patterns:
- “It works on my network but not theirs”
- “Online DNS checkers say it’s wrong”
- “nslookup shows a different IP than the browser uses”

In reality:
- the answers are correct for their respective views
- tools are querying from different contexts
- DNS is behaving exactly as configured

Troubleshooting DNS without knowing which view you are in is unreliable.

---

## Example: The Online DNS Checker Trap

You configure split-horizon DNS for:
service.superawesomehomelabdomain.com.

From inside your network:
- resolves to 10.10.1.20
- works as expected

From an online DNS checker:
- resolves to 203.0.113.20

Incorrect conclusion:
“DNS is inconsistent or broken.”

Correct conclusion:
- the checker is external
- it sees the external view
- the difference is intentional

The tool is not wrong. The assumption is.

---

## How Recursion Interacts with Views

Views apply at the **recursive resolver**, not the client.

Important implications:
- stub resolvers have no awareness of views
- recursion occurs entirely within a selected view
- cached answers are scoped to that view

This means:
- one resolver may maintain multiple caches
- answers from one view must not leak into another
- view selection must happen before caching

---

## Example: View-Scoped Recursion

An internal client queries:
db.superawesomehomelabdomain.com.

The resolver:
- matches the query to the internal view
- resolves the name to 10.10.1.30
- caches the result for the internal view only

An external query for the same name:
- matches the external view
- resolves to a public address or proxy
- uses a separate cache

If view isolation fails:
- clients receive unreachable addresses
- DNS appears intermittently broken
- failures are hard to reproduce

---

## Split Horizon as an Intentional Trade-Off

Split-horizon DNS trades:
- global consistency

for:
- correct routing
- security boundaries
- operational simplicity

It intentionally breaks the idea of a single DNS truth in favour of **contextual correctness**.

## 11. DNS in Modern Systems

Modern operating systems still rely on DNS as heavily as ever — but they increasingly **hide it behind abstraction layers**. 
DNS has moved *up the stack*, away from explicit configuration and visible packets, and into OS services and applications themselves.

This improves reliability for users.
It often reduces visibility for administrators.

---

## systemd-resolved

On many modern Linux systems, DNS resolution is handled by **systemd-resolved**.

Rather than applications talking directly to a configured resolver:
- applications ask the OS to resolve a name
- the OS decides *how* to do that
- DNS configuration becomes contextual and dynamic

systemd-resolved may:
- listen on a local stub address (commonly 127.0.0.53)
- choose different upstream resolvers per interface
- apply split DNS automatically
- cache results internally
- integrate with VPNs and network managers

From the application’s point of view:
- DNS “just works”
- resolver selection is opaque

From the administrator’s point of view:
- `/etc/resolv.conf` no longer tells the full story
- packet captures may not match expectations
- resolution paths depend on runtime state

DNS becomes a **service**, not a file.

---

## Stub Resolvers (Revisited)

Modern systems still use **stub resolvers**, but their role has narrowed.

A stub resolver today:
- forwards queries to a local resolver service
- does not know or care about upstream DNS
- has no visibility into recursion, views, or transport

The difference is *where the stub points*:
- previously → directly to a network resolver
- now → to a local OS-managed intermediary

This extra hop:
- improves flexibility
- but obscures the resolution path

---

## DNS over TLS and DNS over HTTPS (DoT / DoH)

Modern systems increasingly support encrypted DNS transports.

- **DNS over TLS (DoT)**  
  DNS queries over an encrypted TLS connection, usually on port 853.

- **DNS over HTTPS (DoH)**  
  DNS queries sent as HTTPS traffic, usually over port 443.

These mechanisms:
- protect DNS from passive observation
- prevent trivial on-path manipulation
- improve privacy on untrusted networks

They do **not** change DNS semantics.
They change *how queries are transported*.

---

## Application-Level Resolvers

Some applications bypass the OS resolver entirely.

Common examples:
- web browsers
- container runtimes
- language runtimes
- embedded clients

These applications may:
- ship with their own DNS logic
- implement DoH independently
- ignore system DNS settings
- cache aggressively in-process

As a result:
- two applications on the same system may resolve the same name differently
- OS-level DNS changes may not affect running applications
- DNS behaviour becomes application-specific

DNS is no longer a single shared subsystem.

---

## Why Modern Systems “Hide” DNS

This shift is intentional.

Reasons include:
- mobility (Wi-Fi, VPNs, roaming)
- privacy concerns
- application resilience
- security hardening
- reduction of user-visible failure modes

Centralising DNS decisions inside the OS allows:
- smarter resolver selection
- better integration with network state
- fewer hard failures for users

The trade-off is transparency.

---

## Why Debugging Feels Harder Now

Traditional DNS debugging assumed:
- a visible resolver address
- a single resolution path
- a shared cache

Modern systems violate all three assumptions.

Common pain points:
- `resolv.conf` is no longer authoritative
- packet captures miss encrypted DNS
- local stub resolvers mask upstream behaviour
- applications bypass OS settings
- caches exist at multiple layers

The result:
- DNS problems feel inconsistent
- symptoms vary by application
- root causes are harder to observe directly

DNS is still deterministic. It is just **layered**.

---

## DNS Has Moved Up the Stack

DNS has evolved from:
- a simple network lookup
into:
- an OS-managed service
- a privacy-sensitive transport
- an application-level concern

Administrators have not lost control — but they have lost **direct visibility**.

Effective debugging now requires understanding:
- which layer is answering
- which resolver is being used
- which cache is involved
- which transport is in play

DNS has not become unreliable. It has become *contextual*.

## 12. Why DNS Feels Unreliable (But Isn’t)

DNS has a reputation for being flaky, inconsistent, or unreliable.
In practice, DNS is **highly deterministic**.

What feels unreliable is usually a mismatch between:
- how people *expect* DNS to behave
- and how DNS is actually designed to work

DNS is predictable once its rules are understood. It's just **non-intuitive**.

---

## DNS Is Deterministic, Not Immediate

DNS does not promise:
- instant updates
- global agreement
- a single, universal answer

DNS promises:
- bounded reuse of answers (TTL)
- delegation-based authority
- eventual convergence

When DNS appears to “change slowly”, it is honouring its contracts.
When DNS appears to “disagree”, it is operating across boundaries.

Nothing is random. Nothing is ad hoc.

---

## How the Pieces Combine

### Caching

Caching means:
- answers are reused until expiry
- resolvers are allowed to “lie” temporarily
- different resolvers may hold different valid answers at the same time

This enables scale.
It also delays visibility of change.

---

### Delegation

Delegation means:
- no single source of global truth
- authority is split across zones
- responsibility changes at boundaries

Resolvers follow signposts, not shortcuts.
Failures often occur at delegation edges, not within zones.

---

### Propagation

DNS changes do not propagate.
They are **rediscovered**.

Each resolver:
- waits for cached data to expire
- re-queries authority
- updates its local view

This produces gradual convergence, not a wave of updates.

---

### Split Horizon

Split-horizon DNS means:
- the same name can have multiple correct answers
- correctness depends on who is asking
- context matters

DNS is allowed to be inconsistent by design
when that inconsistency reflects routing or security reality.

---

### Client Behaviour

Modern systems:
- hide DNS behind OS services
- add application-level resolvers
- encrypt transport
- introduce multiple caches

As a result:
- different applications may resolve differently
- tools may observe only part of the picture
- debugging feels indirect

DNS has not changed unpredictably. The **number of layers has increased**.

---

## Why Troubleshooting Often Goes Wrong

DNS troubleshooting fails when people assume:
- one resolver
- one cache
- one answer
- one truth

In reality, DNS involves:
- multiple caches
- multiple authorities
- multiple views
- multiple decision points

Looking from the wrong vantage point produces misleading conclusions.

The DNS is not lying. The observer is missing context (usually).

---

## DNS as a System of Contracts

DNS works because it is contractual:

- TTLs define reuse
- delegation defines authority
- views define context
- transports define delivery
- resolvers define responsibility

Each component follows its rules independently.
The system works because those rules compose.

---

## Reframing DNS Reliability

DNS is reliable because:
- it tolerates failure
- it absorbs inconsistency
- it favours availability
- it converges over time

What feels like unreliability is usually:
- impatience
- hidden caches
- mismatched tools
- incorrect assumptions

Once the mental model aligns with the design, DNS stops being mysterious and starts being dependable.

---

## Final Thought

DNS does not fail silently.
It fails *predictably* — just not always visibly.

Understanding DNS is not about memorising commands.
It is about understanding **where you are standing** when you ask the question.

Once you know that, DNS behaves exactly as expected.

## Appendix A — Terminology Reference

**Resolver**  
A system that answers DNS queries on behalf of clients.  
Resolvers may perform recursion, cache results, and follow delegations to authoritative servers.

**Authoritative Server**  
A DNS server that publishes definitive data for one or more zones.  
It answers only for zones it owns and does not perform recursion.

**Recursion**  
The act of taking responsibility for resolving a name fully by following delegations until a final answer is obtained.  
Recursion is performed by resolvers, not authoritative servers.

**Delegation**  
The act of a parent zone indicating which servers are authoritative for a child zone.  
Delegation transfers responsibility, not data.

**Zone**  
An administrative portion of the DNS namespace served by a specific set of authoritative servers.  
A zone represents a boundary of authority, not merely a domain name.

**TTL (Time To Live)**  
The maximum duration a DNS record may be cached before it must be discarded.  
TTL is authoritative metadata and forms a caching contract.

**Negative Caching**  
The caching of non-existence or no-data responses (e.g. NXDOMAIN).  
Negative answers are cached according to rules defined in the zone’s SOA record.

---

## Appendix B — RFC Pointers (Optional)

- **RFC 1034** — Domain Names: Concepts and Facilities  
- **RFC 1035** — Domain Names: Implementation and Specification  
- **RFC 2181** — Clarifications to the DNS Specification  
- **RFC 2308** — Negative Caching of DNS Queries  
- **RFC 4033** — DNS Security Introduction and Requirements  
- **RFC 4034** — DNSSEC Resource Records  
- **RFC 4035** — DNSSEC Protocol

# To Do

- Reverse DNS (PTR records, in-addr.arpa / ip6.arpa, and delegation models)
- DNSSEC operational considerations (key rollover, failure modes, and recovery)
- EDNS(0) and capability negotiation in modern DNS
- Anycast and its role in authoritative DNS scalability
- Vendor- and implementation-specific behaviour (BIND, Unbound, Windows DNS, Cloud providers)
- Practical troubleshooting workflows and tool usage
- Environment-specific DNS patterns (cloud-native, hybrid, container platforms)

