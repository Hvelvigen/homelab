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

