# How Service Patterns Should Be Structured
Service patterns define **how individual services run** on a Host Service Layer.  
They are atomic, reusable components that recipes can combine in different ways.

A service pattern does not configure the host — it configures the workload.

---

# 1. Purpose of a Service Pattern

A service pattern answers:

- *What is this service?*  
- *How should it be deployed?*  
- *Where does its data live?*  
- *What does it depend on?*  
- *How is it accessed?*  

By keeping these consistent across the repository, you gain:

- clarity  
- easy reuse  
- predictable updates  
- minimal duplication across recipes  

Service patterns stay focused and repeatable.

---

# 2. What a Good Service Pattern Includes

Each pattern should follow this structure:

---

## **2.1 Purpose**
A short, clear explanation of what the service does and why someone might deploy it.

---

## **2.2 Prerequisites**
Requirements before deploying this service, such as:

- the Host Service Layer required (e.g. Docker Host)  
- any networking assumptions  
- dependent patterns (e.g. InfluxDB before Grafana)  
- expected hardware profile  

Keep this tight and avoid duplication.

---

## **2.3 Architecture / Behaviour**
A high-level overview of how the service works:

- what components it uses  
- protocols or ports  
- internal/external endpoints  
- expected data flow  

No deep troubleshooting here — only the conceptual model.

---

## **2.4 Directory & Data Structure**
Document:

- persistent data paths  
- config file locations  
- volume mounts  
- logs  

The goal is predictable layouts across all patterns.

Example:

```
/srv/docker/it-tools/compose.yaml
/srv/docker/it-tools/data/
/srv/docker/it-tools/config/
```

---

## **2.5 Deployment Instructions**
The minimal steps needed to deploy the service, such as:

- copying the compose file  
- bringing it up  
- verifying the container  
- initial login or onboarding steps  

Avoid embedding full explanations from other guides.

---

## **2.6 Access & Verification**
Document what “working” looks like:

- URLs or ports to test  
- expected login pages  
- CLI commands, APIs, or dashboards  
- health indicators  

This section is critical for recipes.

---

## **2.7 Update Behaviour**
Describe:

- how updates should be applied  
- whether breaking changes are common  
- which files should be backed up  
- specific notes on upgrade sequencing  

---

## **2.8 Integration Notes (Optional)**
If the service often pairs with others, mention:

- reverse proxy compatibility  
- dependent services  
- common stacks  

This keeps patterns discoverable without bundling recipes.

---

# 3. What Service Patterns Must Not Do

They must not:

- alter the Host Service Layer  
- change OS configuration  
- redefine directory structures globally  
- include deep troubleshooting  
- embed unrelated services  

They stay focused on **one service only**.

---

# 4. Why Service Patterns Matter

They ensure:

- each service is documented once  
- recipes never duplicate instructions  
- upgrades stay consistent  
- automation (future service-builder UI) becomes possible  
- users can mix and match services confidently  

As the repository grows, patterns prevent chaos.

---

# 5. Where Service Patterns Live

Service patterns live under:

```
systems/<platform>/<role>/patterns/<service-name>.md
```

Examples:

```
systems/linux/docker/patterns/it-tools.md
systems/linux/docker/patterns/gitea.md
systems/linux/docker/patterns/uptime-kuma.md
systems/home-assistant/core/patterns/mqtt.md
```

---

This document defines how service patterns should be structured to stay reusable, scalable, and consistent across the Hvelvigen repository.
