## Active Directory–Integrated DNS

Active Directory does not merely *use* DNS — it is **structurally dependent** on it.  
Without DNS behaving correctly, Active Directory cannot locate itself, replicate, or function as a directory at all.

AD-integrated DNS should be understood as a **directory dependency**, not a Windows feature.

---

## Why Active Directory Requires DNS

Active Directory is a **distributed directory**.
Clients and domain controllers must constantly discover:

- domain controllers
- global catalog servers
- authentication services
- replication partners

Hard-coding this information would be brittle and unscalable.
DNS provides the discovery fabric AD depends on.

If DNS is wrong:
- authentication fails
- logons slow or break
- replication stalls
- the directory fragments

Active Directory assumes DNS is:
- authoritative
- consistent
- dynamically updateable

If those assumptions are violated, AD degrades rapidly.

---

## How Active Directory Uses SRV Records

AD relies heavily on **SRV records** to publish service locations.

Rather than asking:
> “What is the IP of a domain controller?”

AD-aware clients ask:
> “Where is the LDAP / Kerberos / GC service for this domain?”

These questions are answered via DNS.

---

### Example: Domain Controller Discovery

Domain: corp.example.com.

_corp._tcp.dc._msdcs.corp.example.com.    SRV    0 100 389 dc1.corp.example.com.
_corp._tcp.dc._msdcs.corp.example.com.    SRV    0 100 389 dc2.corp.example.com.

Meaning:
- multiple domain controllers exist
- all provide the same directory services
- clients can choose any suitable DC
- no DC is hard-coded

This allows:
- DCs to be added or removed
- services to move
- load to distribute naturally

DNS is the directory’s locator service.

---

## Secure Dynamic Updates

Active Directory requires DNS records to be:
- created automatically
- updated as systems change
- protected from unauthorised modification

This is achieved through **secure dynamic updates**.

Semantics:
- domain-joined systems can update their own records
- domain controllers update service records
- updates are authenticated and authorised
- arbitrary systems cannot overwrite records

This keeps DNS:
- accurate
- current
- resistant to spoofing

Manual DNS management does not scale for AD environments.

---

## AD Zones Stored in Active Directory

In AD-integrated DNS:
- DNS zones are stored **inside the directory**
- zone data replicates using AD replication
- no single DNS master exists

This changes the model entirely.

There is:
- no primary DNS server
- no AXFR/IXFR between DNS servers
- no single point of update

DNS becomes **multi-master**, just like AD itself.

---

## Multi-Master Implications

Because DNS data is stored in AD:

- any domain controller hosting DNS can accept updates
- changes replicate automatically via AD
- DNS consistency depends on AD health

This provides:
- high availability
- simplified management
- tight integration

But it also means:
- DNS failures often indicate directory replication issues
- DNS problems are rarely “just DNS” in AD environments

---

## Example: Multi-Master DNS in Practice

You update a DNS record on DC1.

What happens:
- the change is written to Active Directory
- AD replication distributes the update
- DC2 and DC3 learn the change automatically
- all DNS servers become consistent over time

There is no zone transfer.
There is no DNS serial number.
The directory handles versioning and replication.

DNS publication is now a directory concern.

---

## Common Failure Modes

Because of the tight coupling, common AD DNS failures include:

- **Broken SRV records**
  - clients cannot locate domain controllers
  - logons fail or slow dramatically

- **Replication issues**
  - DNS records differ between DCs
  - clients see inconsistent directory state

- **Dynamic update failures**
  - machines cannot register records
  - stale records accumulate
  - name resolution becomes misleading

- **Using non-AD-aware DNS**
  - SRV records missing or incorrect
  - secure updates unavailable
  - directory assumptions violated

These failures often present as:
> “Active Directory is broken”

When the root cause is DNS.

---

## Active Directory DNS as a Dependency

Active Directory treats DNS as:
- a discovery mechanism
- a publication system
- a replication substrate

Not as a convenience.

If DNS does not:
- allow secure updates
- publish SRV records correctly
- replicate consistently

Then Active Directory cannot function reliably.

Understanding AD DNS means understanding that **the directory is built on top of DNS, not alongside it**.
