# Minor Risk — Overlap Between Beginner Linux Guides

There is intentional conceptual overlap between the following beginner-level documents:

- **Understanding Bash on Ubuntu**  
- **Linux Terminal Cheatsheet — Core Terminal Skills**  
- **Working with Files & Directories — Beginner Guide**

Each of these explains foundational topics such as navigating the filesystem, reading and editing files, and understanding basic command patterns.  
This overlap improves approachability for beginners, but it introduces a **maintenance risk**.

---

## Risk  
Updates to behaviour, examples, or recommended patterns may require changes in **three separate files**.  
If only one is updated, the others may fall out of sync over time.

---

## Impact  
- Inconsistencies between guides  
- Confusion for new users when examples differ  
- Additional maintenance load  
- Harder long-term refactoring of foundational material  

---

## Mitigation  
- Treat **Understanding Bash** as the *conceptual foundation* (keep stable).  
- Treat **Core Terminal Skills** as the *quick-reference command layer*.  
- Treat **Working with Files & Directories** as the *practical walkthrough*.  
- When updating command behaviours or navigation patterns, review all three together.  
- Use cross-linking to reduce duplication.

---

## Acceptability  
This is a **low-severity, low-probability** maintenance risk.  
The overlap is intentional and beneficial — it simply requires occasional housekeeping to avoid drift.
