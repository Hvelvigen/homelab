# Hvelvigen Folder Structure  
*A canonical definition of how runtime and service data are arranged on Linux hosts within Hvelvigen.*

This document defines the **official, enforceable folder structure** used by all Host Service Layers (HSLs), Service Patterns (SPs), and Recipes in Hvelvigen.

The goal:

- predictable  
- rebuildable  
- automation-friendly  
- future-proof  

Everything here is **design law**, unless an HSL explicitly defines an override.

---

# 1. Disk Layout Philosophy

All Linux-based hosts in Hvelvigen — regardless of purpose — follow the same foundational rule:

```
/srv is always the root of the secondary disk (sdb)
```


This keeps runtime data, service data, logs, and templates **off the system disk**, allowing:

- OS rebuilds without losing configuration  
- consistent paths across hosts  
- clean mapping for automation and backups

Nothing meaningful should ever write to `/var/lib/<something>` unless upstream tooling gives you no alternative.

---

# 2. Naming Conventions

To keep everything readable and consistent:

- **lowercase always**  
- **hyphens preferred**  
- **numbers allowed when necessary** (e.g. for High Availability clusters or numbered nodes)  
- underscores allowed **only** for:
  - `/meta` in the repo  
  - `_templates` folders on the host

Examples:

```
correct: /srv/host-docker
correct: /srv/docker/it-tools
correct: /srv/host-ha-1
wrong: /srv/HostDocker
wrong: /srv/hostDocker
wrong: /srv/ITTools
```

---

# 3. Host Service Layers (HSL)

An HSL defines what the *host machine* is — its role and runtime.

HSL paths always follow:

```
/srv/host-<role>
```

Examples:

```
/srv/host-docker
/srv/host-reverse-proxy
/srv/host-python
/srv/host-ha # e.g. HA node support tooling
```

### 3.1 Subfolders

HSLs may define their own internal structure, but this document provides **recommended, non-mandatory** examples:

```
/srv/host-docker/
config/
data/
logs/
_templates/
```

> HSL documents remain canonical for their own substructure.  
> This file only sets the global expectations.

### 3.2 Multiple HSL Roles Per Host

Hvelvigen discourages multi-role hosts — simplicity wins — but it's the readers call.

A valid example:

```
/srv/host-docker
/srv/host-python
```

Each HSL remains isolated and self-contained.

---

# 4. Service Patterns (SP)

Service Patterns define *individual services* running on an HSL.

They always live under the **role they depend on**:

```
/srv/<hsl-role>/<service>
```

For a Docker host:

```
/srv/docker/it-tools
/srv/docker/node-red
/srv/docker/gitea
```

For a Python runtime host:

```
/srv/python/home-automation-helper
```

### 4.1 Recommended Internal Layout (Flexible)

Service Patterns should *prefer* but do not have to strictly enforce:

```
/srv/docker/it-tools/
compose.yaml
config/
data/
logs/
...
```

Patterns stay free to adapt for upstream requirements.

---

# 5. Templates

Templates follow a simple rule:

Templates belong to the Host Service Layer.

Each HSL may include:

```
/srv/host-<role>/_templates/
```

This keeps patterns stateless and avoids mixing host logic with service logic.

Examples:

```
/srv/host-docker/_templates/
compose-base.yaml
env-defaults
```

---

# 6. Logs

Runtime and service logs must not clutter service directories.

All logs that do not explicitly belong to a service pattern are written to:

```
/srv/logs
```

Examples:

```
/srv/logs/docker-host.log
/srv/logs/reverse-proxy-access.log
```

Service-level logs stay with the service:

```
/srv/docker/it-tools/logs/
```

---

# 7. Protected Zones & Immutability

To maintain clean automation boundaries, Hvelvigen enforces these immutability rules:

### 7.1 Protected (Automation Must Not Modify)

```
/srv/host-<role>/config
/srv/host-<role>/_templates
```

### 7.2 Automation-Safe to Modify

```
/srv/<hsl-role>/<service>/data
/srv/<hsl-role>/<service>/config
/srv/<hsl-role>/<service>/logs
```

### 7.3 Reader-Owned Experimental Space

If a reader wants a freeform area:

```
/srv/local/
```

This entire folder is intentionally ignored by automation.

---

# 8. Complete Example — Docker Host Running Multiple Services
```
  /srv/
  ├── host-docker/
  │ ├── config/
  │ ├── data/
  │ ├── logs/
  │ └── _templates/
  │
  ├── docker/
  │ ├── it-tools/
  │ │ ├── compose.yaml
  │ │ ├── config/
  │ │ ├── data/
  │ │ └── logs/
  │ │
  │ ├── gitea/
  │ │ ├── compose.yaml
  │ │ ├── data/
  │ │ └── logs/
  │ │
  │ └── uptime-kuma/
  │ ├── compose.yaml
  │ ├── data/
  │ └── logs/
  │
  ├── logs/
  └── local/
```
---

# 9. Summary

- `/srv` is always the runtime root of sdb.  
- HSLs define host roles under `/srv/host-<role>`.  
- Service Patterns inherit from their HSL and live under `/srv/<hsl-role>/<service>`.  
- `_templates` are HSL-level only.  
- Logs are centralised in `/srv/logs` unless service-specific.  
- Everything is lowercase with hyphens unless explicitly `_template`.  
- Multi-role hosts are allowed but not encouraged.  
- This document is the canonical reference for all folder layout decisions across the repository.
