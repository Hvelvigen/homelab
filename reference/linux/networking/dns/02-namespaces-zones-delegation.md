## Namespaces, Zones, and Delegation

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

The dot at the end (`.`) is the **root label**. 

Most tools and humans omit it, but DNS itself treats fully-qualified names as rooted at `.`.

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
