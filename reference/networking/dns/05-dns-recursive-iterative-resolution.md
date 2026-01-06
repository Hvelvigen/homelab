## Recursive vs Iterative Resolution

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
