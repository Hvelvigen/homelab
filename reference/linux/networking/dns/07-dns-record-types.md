## DNS Record Types

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
