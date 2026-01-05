## DNS in Modern Systems

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
