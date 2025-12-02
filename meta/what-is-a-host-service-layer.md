# What Is a Host Service Layer?
The **Host Service Layer** is where a generic, hardened operating system becomes a **purpose-built appliance**.

It defines the *role* of the device — what it is, what it does, and what workloads it is designed to host.

A Host Service Layer does not replace OS hardening or service patterns.  
Instead, it forms the bridge between the two.

---

# 1. Purpose of the Host Service Layer

This layer turns a clean, secured OS into a system with a clear identity and operational role, such as:

- a Docker Service Host  
- a reverse proxy appliance  
- a Home Assistant core node  
- a storage / archive server  
- a Proxmox worker VM  
- a specialised backend service node  

The goal is to define the platform *onto which* service patterns will be deployed.

---

# 2. What the Host Service Layer Includes

A Host Service Layer guide provides:

### **2.1 Directory Structure**
- where persistent data lives  
- where application runtimes sit  
- where configs should be placed  
- a consistent internal layout (e.g. `/srv/docker/<service>`)

### **2.2 Dependencies & Runtimes**
- Docker / Podman / container runtimes  
- Python environments  
- reverse-proxy tooling  
- storage drivers  
- VM agents or hypervisor integration  
- anything required before services can run

### **2.3 Role-Specific Configuration**
- tuning, limits, kernel settings (when role-specific)  
- update strategies  
- log retention relevant for the role  
- service behaviour such as restarts or auto-heal logic

### **2.4 Integration Points**
- how the host interacts with the network  
- DNS expectations  
- proxy behaviour  
- storage or backup locations  
- how the role fits into wider recipes

This layer is **not** service-specific — it prepares the host, not the applications.

---

# 3. What the Host Service Layer Does *Not* Include

It does **not**:

- repeat OS hardening  
- deploy applications  
- define service-specific data stores  
- provide deep troubleshooting  
- set global network design  

Those belong in OS configuration, service patterns, and reference guides.

---

# 4. Why This Layer Exists

The Host Service Layer allows Hvelvigen to:

- keep OS baselines clean and generic  
- avoid repeating Docker/Proxy/HA/Storage setup in every recipe  
- ensure consistent directory structure and behaviour across the repo  
- scale to hundreds of appliances and service stacks  
- make service patterns completely reusable

It is the pivot where a hardened OS becomes a functional platform.

---

# 5. How It Fits into a Recipe

A typical recipe flow:

1. **Base OS** → Installed and booting  
2. **Post-Installation Configuration** → Secured OS  
3. **Host Service Layer** → OS becomes a defined appliance  
4. **Service Patterns** → Add workloads  
5. **Validation** → Confirm functionality and document the final state

The Host Service Layer is where “this node becomes useful”.

---

# 6. Where Host Service Layer Guides Live

Host Service Layer documents are stored under:

```
systems/<platform>/<role>-service-host.md
```

Examples:

```
systems/linux/docker/docker-service-host.md
systems/linux/reverse-proxy/proxy-service-host.md
systems/home-assistant/core/ha-core-service-host.md
systems/hypervisors/proxmox/proxmox-worker-service-host.md
```

---

This document defines the purpose, behaviour, and expectations of the Host Service Layer within the Hvelvigen ecosystem.
