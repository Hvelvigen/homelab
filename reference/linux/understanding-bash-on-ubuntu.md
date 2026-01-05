# Understanding Bash on Ubuntu  
*A practical primer for working in the Ubuntu Server shell.*

Bash is the default command-line shell on Ubuntu Server 24.04.  
It’s the environment you’ll use to install software, update the system, edit configuration files, and manage services across your homelab.

Most guides in Hvelvigen assume you can move around the filesystem, run commands, and make simple edits — this document fills in that foundation.

---

## Repository Location

```
reference/
└── linux/
    └── understanding-bash-on-ubuntu.md   ← you are here
```

## 1. What Bash Is

Bash (Bourne Again Shell) is:

- a command interpreter  
- a simple scripting language  
- the standard shell for Ubuntu Server  

When you log in via SSH or the console, you’re using Bash.

---

## 2. Anatomy of a Command

At its core, a Bash command is just three things in a line:

a command,
some optional flags,
and whatever you want it to act on.

```
command   options   arguments
```

That’s it.
Everything you’ll type follows this same pattern.

Let’s use a real example:

```
ls -l /etc
```

Here’s what’s actually happening:

ls — the command.
You’re asking Bash to list what’s in a directory.

-l — an option (or flag).
This tweaks the behaviour.
In this case, -l means “give me the detailed view — permissions, sizes, timestamps, the works.”

/etc — the argument.
This is the thing you’re pointing the command at.
You could aim ls at any folder, but here you’re asking it to look specifically inside /etc.

So the whole command, in plain English, is:

“List the contents of /etc, but show me the detailed version.”

Once you understand this pattern —
command → options → arguments —
you can read almost any Bash command and immediately know what it’s trying to do.

And you’ll see this pattern everywhere in Hvelvigen guides.

---

## 3. Navigating the Filesystem

Bash works a lot like walking through folders on your PC — you’re just doing it with commands instead of windows and icons.
These are the ones you’ll use constantly:

```
pwd            # show current directory
ls             # list files
ls -l          # detailed list
cd /path       # move to a directory
cd ..          # up one level
cd             # go to your home directory
```

A couple of symbols you’ll see all the time:

/  → the very top of the filesystem (the "root" of everything)

~  → your home directory (your personal starting point)

---

## 4. Editing Files

Sooner or later (**spoiler**: It's sooner) you’ll need to tweak a config file.
Ubuntu Server ships with `nano`, which is perfect for this — simple, predictable, less of a headache than `vi` and not out to impress anyone.

To open a file:

```
nano /etc/ssh/sshd_config
```

Once you’re inside nano, you only need three controls:

  - Ctrl+O — save your changes
  - Enter (or Y, depending on the prompt) — confirm the save
  - Ctrl+X — exit the editor

That really is it. For 99% of homelab configuration work, nano is all you’ll ever need.

---

## 5. Running Commands as Root

Some tasks need administrator privileges — things like installing packages, restarting services, or changing system settings.
Ubuntu handles this through `sudo`, which lets you run a single command with elevated rights when you need to.

Examples:

```
sudo apt-get update
sudo apt-get upgrade -y
sudo systemctl restart ssh
```

A good rule of thumb:
Use sudo only when the system asks for it or when you know the action genuinely requires admin access.

If a command needs higher privileges and you run it without sudo, Bash will make it very clear. You won’t break anything — you’ll just be reminded to run it with the right permissions.
*(Tip: press ↑ to recall the command, move to the start of the line, and add 'sudo ' — including the space — before rerunning it.)*

---

## 6. Installing and Updating Software (APT)

Ubuntu uses APT for installing and updating software.
You’ll use it a lot — updates, new packages, removing things you no longer need.
The patterns stay the same:

```
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install <package>
sudo apt-get remove <package>
```

This is how you install things like `btop`, Docker, OpenSSH Server, and other baseline tools.

---

## 7. Reading Output

Bash will always tell you what happened — you just need to know how to read what it’s saying.

Most commands produce three types of output:

- normal output — what the command actually wanted to show you
- warnings — something wasn’t ideal, but the command still ran
- errors — the command couldn’t complete what you asked

When something goes wrong, don’t just look at the last line.
Scroll up a little and read the full message.
Linux usually explains exactly what the problem was — missing packages, wrong paths, permission issues, typos, or a service refusing to start.

A good habit is to ask yourself:

- What was the command trying to do?
- What part of the output tells me why it didn’t?
- Does the message mention a file, a service, or a permission?

Most fixes become obvious once you understand which part of the command failed.
This style of reading output is one of the best skills you’ll build — and it makes every guide in Hvelvigen easier to troubleshoot if something goes wrong.

---

## 8. Pipes and Redirection

Sometimes you’ll want to take the output of one command and feed it into another, or save it to a file so you can review it later.
Bash makes this very easy.

### Using a pipe

A pipe `("|")` takes the output of one command and passes it directly into the next:

```
ls -l | grep ".log"
```

This lists everything in the directory, then filters the list so you only see items containing .log.
Pipes are perfect for narrowing things down when the output is noisy.

### Redirecting output to a file

You can also send a command’s output into a file instead of the screen:

```
command > file.txt      # overwrite the file
command >> file.txt     # append to the file
```

Great for capturing logs, saving command results, or keeping a record of what happened without scrolling back through the terminal.

---

## 9. Useful Keyboard Shortcuts

A few shortcuts make Bash far smoother to use, especially when you’re working quickly or running repeated commands. 

These are the ones you’ll rely on constantly:

- `Ctrl+C` — stop whatever’s currently running
  This is useful when a command is taking too long, you started the wrong thing, or you realise you’ve made a typo. 
  It cleanly interrupts the process and drops you back to the prompt.

- `Ctrl+R` — search your command history
  Type a few letters and Bash will pull up matching commands you’ve run before.
  This is brilliant for long commands, paths, or anything involving multiple flags — no need to retype something you’ve already used.

- `Tab` — autocomplete paths and filenames
  Tap Tab once to complete a unique match, or twice to see all possible matches.
  It reduces typos dramatically and makes moving around the filesystem far quicker.

Together, these shortcuts take a lot of the friction out of the command line.
Once they become second nature, everything else in Bash feels more predictable and much faster.

More shortcuts can be found in the keyboard-shortcuts reference guide.

---

## 10. Where Things Live (Practical Map)

Knowing where Ubuntu keeps things will save you a lot of digging.
Most of the time you’ll be working with just a handful of key locations:

- `/etc/`          System-wide configuration files — everything from SSH settings
               to service configs. If you’re editing behaviour, it’s probably here.

- `/var/log/`      Logs for services and the system. This is where you look when
               something isn’t behaving and you want to know why.

- `/usr/bin/`      Installed programs and command-line tools. When you run a command
               like `ls`, it’s coming from somewhere in here.

- `/home/<user>/`  Your personal space — scripts, notes, downloads, anything tied
               to your account.

- `/srv/`          Service data and persistent application files. In Hvelvigen,
               this is where your Docker containers keep their configs and data.

These folders appear everywhere across your homelab guides — installation, SSH setup, Docker work, service patterns — I'd recommend getting familiar with them.

---

## 11. Bash in Hvelvigen Workflows

Bash is the command-line you’ll use for Ubuntu Server and other Linux-based work in your homelab. 
Whenever you’re building or maintaining anything on the Linux side, this is likely where the action happens.

You’ll use Bash when:

- installing and configuring Ubuntu Server
- running post-install tasks
- setting up SSH access and security
- preparing Docker hosts and managing containers
- reviewing logs and checking system health
- installing updates and packages
- editing configuration files for Linux services
- following any Linux-focused recipe or baseline in Hvelvigen

If a guide in the repo talks about Ubuntu or Docker on Linux, it’s probably Bash you’ll be using.

---

## 12. Safety Tips

A few habits go a long way when you’re working in the shell:

- Pause if a command looks destructive.
  If it includes rm, mv, or touches system paths, take a second look.
  A quick re-read is cheaper than an unintended rebuild.

- Don’t guess.
  If something feels off, check the path, the spelling, or the command options.
  Most mistakes come from typing faster than you’re thinking.

- Keep a console or hypervisor window open when changing SSH settings.
  If you misconfigure SSH, you’ll be very glad you left yourself another way back in.

- Use sudo intentionally, not automatically.
  Commands that need elevated rights will tell you.
  If a non-privileged action fails, the error message will make it clear.

Common danger signs to watch for:

- rm -r or rm -rf (recursive deletion — powerful and unforgiving)
- mv with broad patterns (easy way to lose track of files)
- commands referencing /* (the entire filesystem — never good news)
- wildcards (*) in paths where you didn’t mean to include everything
- editing files directly in /etc without making a quick copy first
- piping commands into sudo without understanding what the full pipeline does

These small habits prevent the majority of “how did that even happen?” moments.

---

## 13. Summary

Bash is the foundation of working with Ubuntu Server — a consistent, predictable way to get things done without hunting through menus or waiting for a UI to load “just a moment” for the tenth time. 

Once you’re comfortable moving around directories, editing configuration files, and using sudo when something actually needs authority (not just because you’re feeling powerful), the rest of the Linux path in Hvelvigen falls neatly into place.

You don’t need deep knowledge or an encyclopaedic memory. 

You just need familiarity — the kind that comes from using the basics regularly until they stop feeling like “commands” and more like “just how you talk to the server.”

Everything in Hvelvigen builds on these simple patterns, because complicated setups are overrated and usually end with future-you muttering “who wrote this?” (**Spoiler**: it was *you*).

This guide gives you the baseline you’ll use everywhere else: SSH, Docker, service configs, logs — it all starts here. With a bit of practice, Bash becomes one of the calmest parts of your homelab. 

It rarely surprises you, it never moves buttons around after an update, and it doesn’t insist on restarting to apply a colour scheme. 
Honestly, it’s quite refreshing.

A few final truths, now that you’ve reached the end:

- If a command runs and nothing dramatic happens, that’s success — Bash   celebrates by doing absolutely nothing noticeable.
- If a command runs and too much happens… well, that’s why we keep console access open and pretend everything is fine until it actually is.
- And if you ever type something and think, “Surely this won’t break anything,” that’s exactly when to take a breath and read it again.

With this foundation, you’re ready for the rest of the Linux side of Hvelvigen — and future-you will genuinely thank you for learning this before diving into the deeper bits.
