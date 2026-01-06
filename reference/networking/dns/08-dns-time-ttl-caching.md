## Time, TTLs, and Caching Semantics

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
