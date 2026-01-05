## Why DNS Feels Unreliable (But Isn’t)

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
