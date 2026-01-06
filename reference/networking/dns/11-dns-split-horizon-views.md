## Split Horizon and Views

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
