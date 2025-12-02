# How Host Service Layers Should Be Structured
A **Host Service Layer** defines the role and behaviour of a device after the OS has been installed and secured.  
It transforms a generic operating system into a **purpose-built appliance** ready to run service patterns.

This document explains how Host Service Layer guides should be structured so they remain consistent, scalable, and reusable across the Hvelvigen repository.

---

# 1. Purpose of a Host Service Layer Guide
A Host Service Layer guide answers the question:

**“What is this device going to be?”**

Examples include:

- a Docker Service Host  
- a reverse proxy appliance  
- a Home Assistant core node  
- a storage node  
- a VM worker for a hypervisor  
- an automation or backend service node  

The guide must define the device’s identity without embedding application-specific behaviour.

---

# 2. Structure of a Host Service Layer Guide

Each guide should follow this consistent structure:

---

## **2.1 Purpose / Role Definition**
A short paragraph describing:

- what the appliance is for  
- why you would choose this role  
- how it fits into a wider homelab design  

Keep this clear and high-level.

---

## **2.2 Prerequisites**
State what must exist before applying this layer:

- OS installed (Base Operating System layer)  
- OS secured (Post-Installation Configuration layer)  
- hardware expectations  
- networking assumptions (e.g. static IP recommended)  
- any platform-specific requirements  

Do **not** repeat OS configuration content here.

---

## **2.3 Architecture & Behaviour**
Describe the conceptual model of this host role:

- what runtime(s) it will provide (Docker, Podman, Proxy, HA Core, etc.)  
- how workloads will execute  
- what system components are involved  
- expected traffic or data flows  

This helps the reader understand *why* the host behaves as it does.

---

## **2.4 Directory Structure**
Document a predictable directory layout, including:

- base directory for this role  
- locations for configs  
- locations for persistent data  
- logs and runtime files  
- any recommended organisational patterns  

Example:

```
/srv/docker/
/srv/docker/_templates/
/srv/<role>/config/
/srv/<role>/data/
```

Consistency here is key for automation, rebuilding, and future recipes.

---

## **2.5 Dependencies & Runtimes**
Define what must be installed for this host role to function, including:

- container runtimes  
- proxy tooling  
- language runtimes (Python, Node, etc.)  
- storage drivers  
- service supervisors  
- VM tools or hypervisor integration agents  

Provide installation steps or point to guides.

This prepares the host for service patterns.

---

## **2.6 Role Configuration**
Configuration specific to the host’s identity, such as:

- tuning relevant to the role  
- system limits  
- file permission patterns  
- systemd services or helpers  
- backup expectations  
- role-appropriate update strategy  

Avoid “service pattern” behaviour here — that belongs elsewhere.

---

## **2.7 Validation**
A checklist confirming this host role is correctly configured.

For example, for a Docker Host:

- Docker engine installed  
- containers run successfully  
- directory structure exists under `/srv/docker`  
- correct network behaviour  
- logs writing to expected locations  

Validation gives the user confidence before they proceed to service patterns.

---

## **2.8 Documentation Requirements**
Explain what the reader should capture:

- versions  
- config deviations  
- hostname / IP address  
- role identity  
- directory and data locations  

This ensures future rebuilding is straightforward.

---

# 3. What Host Service Layer Guides Must Not Do

They must **not**:

- repeat OS installation steps  
- repeat OS hardening or base configuration  
- embed service pattern instructions  
- include deep troubleshooting  
- redefine global standards (naming, security, logging)  

They stay strictly focused on the device’s **role**, not the services running on it.

---

# 4. Why This Structure Matters

Defining Host Service Layers consistently:

- keeps recipes clean and predictable  
- avoids duplication across dozens of stacks  
- enables service patterns to be truly reusable  
- supports future automation or “build your stack” tooling  
- maintains clarity even as the repository grows large  

This structure ensures Hvelvigen remains scalable, understandable, and maintainable in the long term.

---

# 5. Where Host Service Layer Guides Live

Host Service Layer guides are located under:

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

This document defines how Host Service Layer guides must be structured to stay consistent across the Hvelvigen repository.



