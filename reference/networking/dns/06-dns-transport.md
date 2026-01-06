## DNS Transport Mechanics

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
