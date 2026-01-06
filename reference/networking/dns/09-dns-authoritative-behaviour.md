## Authoritative DNS Behaviour

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
