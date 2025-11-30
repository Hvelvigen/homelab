# Working with Files & Directories — Absolute Beginner Guide

A gentle, practical introduction to how Linux organises files, how to navigate the filesystem without getting lost, and how to safely view and edit configuration files.

This guide sits alongside the Core Terminal Skills cheatsheet and explains the **concepts** behind those commands.

---

## 1. What This Guide Teaches

By the end, you’ll understand:

- what files and directories actually are
- how the Linux filesystem is organised
- how to move around without breaking anything
- how to inspect and edit configuration files safely
- how to avoid accidental deletions
- where homelab configs usually live in Hvelvigen

This guide is beginner‑friendly — with no assumptions, and no jargon.

---

## 2. Understanding the Linux Filesystem

Linux organises everything into a single tree that starts at the **root**:

```
/
├── home/          # your personal files and folders
├── etc/           # system and service configuration files
├── var/           # logs, caches, and variable data
├── srv/           # service data (your Docker & app folders live here in Hvelvigen)
├── usr/bin/       # programs and commands
├── tmp/           # temporary files
└── ...            # many other system paths
```

### Key locations you will work with

- **`/home/<youruser>`** — your personal space  
- **`/etc`** — configuration files for services (SSH, Docker daemon, system settings)
- **`/var/log`** — logs you’ll inspect during troubleshooting
- **`/srv/docker/...`** — where Hvelvigen places container data and configs
- **`/tmp`** — safe scratch area (automatically cleaned)

Remember:  
You can’t “double‑click” in Linux — the terminal *is* how you navigate.

---

## 3. Paths: Absolute vs Relative

A *path* is just an address to a file or folder.

### Absolute paths  
Start with `/`:

```
/etc/ssh/sshd_config
/srv/docker/it-tools/
```

They always mean the same thing, no matter where you currently are.

### Relative paths  
Do **not** start with `/`:

```
logs/
backup.sh
../configs/
```

They’re relative to your *current* directory.

### Useful shorthand

```
/   → root of the entire filesystem  
~   → your home directory  
..  → go up one level  
.   → current directory  
```

---

## 4. Moving Around Without Getting Lost

These commands tell you where you are and how to navigate:

```
pwd                 # show your current directory
cd /etc             # go to /etc
cd ~/Documents      # go to Documents in your home folder
cd ..               # go up one level
cd                  # return home
```

**Tab completion** is your friend:  
type the first few letters of a folder → press `Tab`.

---

## 5. Seeing What’s Inside a Folder

```
ls                  # list files
ls -l               # detailed list
ls -a               # include hidden files (start with .)
```

Hidden files are common for config and SSH keys:

```
~/.ssh/
~/.bashrc
```

---

## 6. Reading Files Safely

Before editing anything, read it first.

```
cat <file>          # print full file
less <file>         # scroll through safely
head <file>         # first 10 lines
tail <file>         # last 10 lines
tail -f <file>      # follow live changes
```

### Why “less” is safer than “cat”  
It prevents you accidentally flooding the screen with a giant log file, and you can always cat later.

Use:

- `Space` / `PageDown` to move down  
- `/` to search  
- `q` to quit  

---

## 7. Editing Files (nano)

Nano is the default editor on Ubuntu Server and ideal for beginners.

```
nano <file>
```

Inside nano:

```
Ctrl+O    save
Enter     confirm filename
Ctrl+X    exit
Ctrl+W    search
Ctrl+K    cut line
Ctrl+U    paste line
```

### Always back up configs before editing

```
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

This single line can save you hours of pain.

---

## 8. Creating, Copying, Moving & Renaming Files

### Create an empty file

```
touch notes.txt
```

### Copy files

```
cp file.txt backup.txt
cp -r /source/folder /destination/
```

### Move or rename (same command!)

```
mv oldname.txt newname.txt
mv file.txt /srv/docker/it-tools/
```

### Delete files and folders

```
rm file.txt
rm -r foldername
```

**Warning:** `rm -r` does not ask for confirmation.  
Always `ls` the directory first so you know exactly what you’re removing.

---

## 9. Common Config Locations in a Homelab

### System-level configs

```
/etc/ssh/sshd_config
/etc/fail2ban/jail.local
/etc/docker/daemon.json
```

### Service data (Hvelvigen convention)

`/srv/docker/<service>/`

Examples:

```
/srv/docker/it-tools/
    ├── compose.yaml
    ├── data/
    └── logs/
```

### Logs

```
/var/log/syslog
/var/log/auth.log
/var/log/<service>/
```

---

## 10. Safe Workflow Habits

- Never edit a config without making a backup  
- Always `less` a file before changing it  
- Document what you changed (see lightweight documentation guide)  
- When unsure, create a copy in your home directory first  
- Use tab‑complete to avoid typos  
- Avoid `sudo rm -r` unless *absolutely certain*

---

## 11. Mini Exercises

These help cement the basics.

1. Navigate to `/etc`, list the files, and return home.  
2. View `/etc/ssh/sshd_config` with `less`.  
3. Make a folder under `~/test`, create a file, then delete it.  
4. Copy `/etc/hosts` into your home directory and inspect it.  
5. Create a backup of a config file using `cp`.

---

## 12. Summary

You now understand the essentials:

- how the filesystem is structured  
- how to navigate confidently  
- how to read and edit configuration files safely  
- where homelab configs and logs usually live  

Pair this with the **Core Terminal Skills** cheatsheet for quick command lookups, and you’ll soon be comfortable working inside Ubuntu Server for any recipe or service pattern.

