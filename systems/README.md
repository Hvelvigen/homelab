# Systems

This directory defines the baselines used across Hvelvigen — or, putting it another way, *“follow me to achieve this outcome.”*  
It contains the complete set of walkthroughs that show you exactly how to reach a known, predictable result, whether you're starting with a blank machine or building on top of an existing Host Service Layer.

A “system” can be:

- an OS baseline  
- a hardened platform  
- a Docker-ready host  
- a Windows environment with WAC configured  
- a Service Pattern being added to a ready HSL  
- or any other component where the end result matters and the steps to get there shouldn’t be invented from memory

If a recipe needs it, the walkthrough lives here.

Each system provides:

- **assumptions**  
  What this baseline expects — hardware, environment, prerequisites, and the bits that can’t be hand-waved.

- **installation or activation approach**  
  How to start the process, whether that’s booting an ISO, enabling a feature, or preparing an existing HSL for the next layer.

- **post-install or post-configuration steps**  
  The shaping work: hardening, defaults, structure, services, and the settings that define the intended outcome.

- **validation steps**  
  Checks that confirm the system is in the correct state, rather than politely insisting it is.

This directory exists for one purpose:  
so you never have to guess how you built something, why it works, or which step future-you forgot to write down.

If a system takes you from start to finish without surprises, excellent. Quiet competence is wildly underrated.
