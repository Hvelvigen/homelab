# Hvelvigen Homelab Baseline

*Ett helt valv för din homelab‑resa*

Hvelvigen is your homelab pick‑and‑mix: a recipe-friendly, modular collection of guides, templates, and patterns you can pull from, whenever you want to build something clean, organised, and not powered by pure chaos.  

Rather than defining one “correct” way to build a system, this repository presents **multiple valid approaches**, allowing you to mix, match, and adapt based on your environment and skill level.

---

## Name (EN_GB)
*(uttalas/pronounced: “VEL-vee-yen”)*

**Hvelvigen** comes from Swedish roots: *hel* (whole, complete) and *valv* (vault, arch), using the older **hv** spelling because, well... honestly, I think it adds a bit of charm.  
The meaning — *det hela valvet* (“the complete vault”) — lines up nicely with what this repo aims to be: a safe, structured place for documentation, templates, and ideas.

## Namn (SV_SE)
**Hvelvigen** kommer från *hel* (hel, fullständig) och *valv* (valv, ark). Den äldre stavningen **hv** ger namnet en tidlös känsla.  
Betydelsen — *det hela valvet* — speglar repoets roll som en trygg plats där struktur, dokumentation och idéer bevaras.

---

## Purpose
Hvelvigens purpose is to teach, instil good practices and give you a clean, flexible baseline for documenting and, hopefully, helping you level up your homelab — one recipe at a time.

Hvelvigen gives you **reliable starting points and repeatable patterns** for building and documenting your homelab — without telling you how your environment *must* be built.

Whether you're running Proxmox or Hyper‑V, Docker or Podman, Plex or Jellyfin, the idea is the same: show good ways of doing things, offer alternatives, and let you choose the flavour that fits your stack.

---

## Scope
This repository includes:

- Modular templates  
- Example configurations for different platforms and tools  
- Alternative implementations for the same workflow  
- Best practices, including:
    - Documentation templates  
    - Naming and structure guidance 
- Optional helper scripts and tools  

This repository does **not** include:

- Personal environment details  
- Cloud-specific config  
- Proprietary diagrams or layouts  

---

## Repository Structure

- **/templates/** — reusable templates for docs, configs, and automation  
- **/systems/** — platform-specific baselines (Proxmox, Hyper‑V, Home Assistant, Docker, Linux, etc.)  
- **/reference/** — command sets, cheat sheets, and best-practice notes  
- **/docs/** — naming, structure, and organisation guidance  
- **/tooling/** — optional helper utilities and scripts  
- **/meta/** — repo governance, roadmap, and evolution  

Pick what you want — nothing demands you use the whole thing.

---

## Included Templates
Templates here are intentionally lightweight:

- Directory structures for services and documentation  
- YAML / JSON / `.env` config templates  
- Markdown templates for notes and guides  
- Optional automation starters (shell, PowerShell, etc.)  

They’re designed to be copy‑paste‑edit‑go. No drama.

---

## How to Use This Baseline
Hvelvigen is designed to be modular and practical. You can:

- take the whole structure  
- combine baselines across multiple platforms  
- copy only the folders you need  
- compare multiple approaches to the same service  

Suggested flow:

1. Browse **/systems** to see the platform baselines.  
2. Pull templates from **/templates** to build your layout.  
3. Check **/reference** for commands and operational notes.  
4. Use **/docs** to keep naming and structure consistent.  
5. Add optional tools from **/tooling** if they help.

You don’t need any specific skill level or existing structure — this repo meets you wherever you’re starting.

---

## Recommended Tooling
<details>
  <summary>Read more ...</summary>
  <br/>
  Useful (but optional):
  
  - Git / GitHub / Gitea  
  - Markdown editors (VSCode)  
  - Docker / Podman / Compose  
  - Platform-neutral monitoring tools  
  - Shell / PowerShell scripting  
</details>

---

## Naming Guidance
<details>
  <summary>Read more ...</summary>
  <br/>
  A few principles behind the naming patterns:

- clarity over cleverness  
- consistency over complexity  
- predictable structure  

Full naming guidance is in **/docs/naming.md**.
</details>

---

## Licence
Licensed under the **GNU GENERAL PUBLIC LICENSE**.  
Adapt it, reuse it, build on it — that’s the whole point.
