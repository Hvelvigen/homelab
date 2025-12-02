# What Is a Recipe?
A recipe in Hvelvigen is a **guided, outcome-driven path** that takes you from an idea to a clean operating system with a fully working appliance or service.  
It does this by combining four separate layers:

1. **Base Operating System** – the operating system exists and boots.  
2. **Post‑Installation Configuration** – the OS is hardened, predictable, and safe.  
3. **Host Service Layer** – the device becomes a purpose-built appliance (e.g. Docker host).  
4. **Service Pattern Layer** – one or more services are deployed on top (e.g. IT-Tools).

A recipe is not a standalone guide; it is a **composition** of guides that produces a validated outcome.

---

# Full Specification

## 1. Purpose of Recipes
Recipes provide a **step-by-step walkthrough** for building a specific, reliable end state such as:

- a fully configured Docker service host  
- a working Home Assistant deployment  
- a reverse-proxy appliance  
- a completed stack made of several service patterns  

They are designed to be repeatable, stable, and understandable even by those who are not experts in the underlying technologies.

Recipes **do not** contain all instructions within themselves.  
Instead, they orchestrate and reference modular guides across the repo.

---

## 2. The Four Layers of a Recipe

A complete recipe draws from four clearly separated layers.  
These layers keep documentation modular, reusable, and scalable as the repository grows.

---

### Layer 1 — Base Operating System 
*“The operating system exists and boots to a login prompt.”*

This layer includes only:

- installation of the base OS  
- completing the installer  
- rebooting to a working login screen  

No security, no configuration, no role definitions.

This layer ensures every recipe starts from the same predictable point.

---

### Layer 2 — Post‑Installation Configuration 
*“The OS is hardened, predictable, and ready for purposeful configuration.”*

This layer applies:

- security baselines  
- SSH key access  
- firewall configuration  
- audit and logging setup  
- basic packages  
- system updates  
- naming consistency  

Critically, this layer remains **agnostic** about how the machine will be used.  
It prepares a clean, safe operating environment — nothing more.

---

### Layer 3 — Host Service Layer  
*“This machine becomes a specific appliance with a defined role.”*

The Host Service Layer transforms a generic, hardened OS into a **purpose-built device**, such as:

- a Docker Service Host  
- a storage node  
- a reverse proxy appliance  
- a Home Assistant core node  
- a VM worker for hypervisors  
- a service aggregator  

This layer defines:

- directory structures  
- runtimes and dependencies  
- role-specific tuning  
- service-host behaviour  
- how workloads will be executed  

This is the **identity** of the device.

The OS remains the same; the device role becomes clear.

---

### Layer 4 — Service Pattern Layer  
*“The individual services that run on the appliance.”*

Each pattern describes:

- what the service does  
- where its data lives  
- how it runs (e.g. Compose, Podman, VM, etc.)  
- how it exposes ports / APIs  
- how it integrates with other services on the host  

Examples:

- IT-Tools  
- Uptime-Kuma  
- Gitea  
- Home Assistant add-ons (Mosquitto, InfluxDB, Node-RED)  
- Reverse-proxy patterns  
- Logging and monitoring stacks  

Service patterns are **atomic and reusable**:  
the same pattern can appear in many recipes.

---

## 3. How Recipes Use These Layers

A recipe guides the user through:

1. **Base Operating System** → the machine exists  
2. **Post‑Installation Configuration** → the machine is secured  
3. **Host Service Layer** → the machine gains its identity  
4. **Service Patterns** → the machine runs meaningful workloads  
5. **Validation & Documentation** → the user ends with a reliable, understood system  

Recipes point to guides for each layer, then return to confirm:

- expected state  
- configuration decisions  
- data locations  
- access endpoints  
- any deviations  

This creates a clean, readable workflow without burying details inside the recipe itself.

---

## 4. What Recipes Are Not

To keep things clear:

- They are **not** all-in-one guides  
- They are **not** deep troubleshooting documents  
- They are **not** replacements for upstream documentation  
- They are **not** OS-specific manuals  

Their sole purpose is to **connect existing guides** into a coherent path a user can follow to produce a validated outcome.

---

## 5. Why Recipes Exist

Recipes allow Hvelvigen to scale massively without duplication.

Because each recipe reuses the same layers:

- OS installation guides remain single-source.  
- Hardening is defined once.  
- Host roles are reusable across dozens of stacks.  
- Service patterns do not need rewriting each time they appear.  

As the repository grows into hundreds of services and patterns, the structure remains stable and predictable.

---

## 6. Where Recipes Live

Recipes are stored under:

```
recipes/
<category>/
<recipe-name>.md
```


Categories might include:

- starter - Useful for beginners or to achieve specific outcomes
- docker  
- home-assistant  
- infrastructure  
- hypervisors  
- storage  
- application-stacks  

This keeps recipes discoverable and grouped by purpose.

---

## 7. The Expected Output of a Recipe

Every recipe must end with:

- a functioning appliance or service  
- documented state  
- a validation checklist the reader can pass  
- clear reference to all layers used  
- no hidden steps or assumptions  

The reader should be able to answer:

- *“Yes, this is working.”*  
- *“Yes, I know where everything lives.”*  
- *“Yes, I can rebuild this.”*

If those are true, the recipe is complete.

---

## 8. Extending Recipes in the Future

Recipes can later be expanded with:

- variants (LAN-only vs proxied, minimal vs extended stacks)  
- optional follow-on recipes  
- choice-based service selection 
- multiple hardware profiles (x64, ARM64, etc.)  

But the core definition remains the same:  
**a curated path to a known, validated outcome.**

---

This document defines the concept, purpose, and structure of recipes within the Hvelvigen repository.  
