# Linux Terminal Cheatsheet — Core Terminal Skills

This section gives you the practical day-to-day commands you’ll use on the Linux side of your homelab.  
It’s written for beginners who already understand tech, but are new to Linux — or anyone who wants a clear, predictable baseline before diving into deeper system guides.

Stopping you searching Google for “that one command I should definitely know by now.”

---

## 1. Terminal Basics

The terminal is a simple prompt waiting for instructions.  
Commands follow a familiar pattern:

```
command   options   arguments
```

Example:

```
ls -l /etc
```

- `ls` — what you want to do  
- `-l` — how you want it formatted  
- `/etc` — where to point it  

---

## 2. Navigation (Moving Around)

Knowing where you are — and where you’re going.

```
pwd              # show your current location
ls               # list files
ls -l            # detailed list (permissions, sizes, timestamps)
ls -a            # include hidden files (those starting with .)
cd <path>        # move to a folder
cd ..            # go up one level
cd               # return to your home directory
```

Useful symbols:

```
/   → the root of the filesystem
~   → your home directory
```

Absolute paths start with `/`.  
Relative paths do not.

---

## 3. Listing & Inspecting Directories

When you want to understand what’s in a folder — or filter it:

```
ls                # basic list
ls -l             # long format
ls -lh            # long format with human-readable sizes
ls -lrt           # sort by oldest first
ls *.log          # list only .log files
```

These appear constantly across the repo when navigating config directories under `/etc` or host service folders under `/srv`.

---

## 4. Creating & Managing Files

Linux keeps things simple.  
If you can move, copy, and delete files confidently, you can manage most of the system.

```
touch <file>                # create an empty file
cp <source> <destination>   # copy a file
cp -r <src> <dest>          # copy a directory
mv <src> <dest>             # move or rename
rm <file>                   # delete a file
rm -r <folder>              # delete a folder and its contents
```

`rm -r` does exactly what you ask — no safety checks.  
Always read the path again before pressing Enter.

---

## 5. Working with Directories

```
mkdir <name>         # create a directory
mkdir -p <path>      # create nested directories
rmdir <folder>       # remove an empty directory
rm -r <folder>       # remove a directory and all contents
```

Most service guides in Hvelvigen use this to create consistent directory structures under `/srv/docker` or similar locations.

---

## 6. Viewing File Contents

```
cat <file>           # print file contents
head <file>          # first 10 lines
tail <file>          # last 10 lines
tail -f <file>       # follow file changes in real time
less <file>          # view a file interactively
```

`less` is your friend when dealing with long files — arrow keys, page up/down, `/` to search, `q` to quit.

---

## 7. Editing Files (nano)

`nano` is the default editor in Ubuntu Server.

```
nano <file>
```

Inside nano:

```
Ctrl+O    save changes
Enter/Y   confirm save
Ctrl+X    exit
Ctrl+W    search inside file
```

---

## 8. Searching Inside Files (grep)

```
grep "text" <file>            # search inside a file
grep -R "text" <folder>       # search recursively in a folder
grep -n "text" <file>         # include line numbers
```

---

## 9. Searching for Files & Commands

```
find /path -name "<pattern>"    # search for files by name
locate <filename>               # fast search (requires updatedb)
which <command>                 # where a command is located
whereis <command>               # broader search
```

---

## 10. Permissions & Ownership

```
drwxr-xr-x   # directory with read/write/execute for owner, read/execute for others
-rw-r--r--   # file readable by everyone, writable only by owner
```

Key commands:

```
ls -l                     # view permissions
chmod 644 <file>          # standard file permissions
chmod 755 <folder>        # standard directory permissions
chown user:group <file>   # change ownership
umask                     # view default creation permissions
```

---

## 11. Users & Groups

```
id                                  # show user + group info
groups                              # list groups
sudo useradd <name>                 # create a user
sudo passwd <name>                  # set password
sudo usermod -aG <group> <user>     # add user to a group
```

---

## 12. sudo & Privilege Management

```
sudo <command>      # run as root
sudo -k             # clear cached credentials
sudo !!             # rerun previous command with sudo
```

---

## 13. Environment Variables

```
echo $PATH              # show PATH
export VAR=value        # set variable temporarily
printenv                # list variables
```

---

## 14. Keyboard Shortcuts & Productivity

```
Ctrl+C      stop current command
Ctrl+R      search history
Ctrl+L      clear screen
Tab         autocomplete
!!          repeat last command
history     show command history
```

---

## 15. Danger Zone (Commands to Respect)

```
rm -rf <path>       # recursive delete — double-check the path
mv *                # wildcard moves — easy to over-match
chmod -R            # recursive permission changes
sudo !!             # useful but risky if the last command was wrong
```

If you ever think “should I run this?” — stop and check.

## 16. Summary
Core terminal skills are the foundation of everything you’ll do on the Linux side of your homelab. 
Once you can move around confidently, read files, edit configs, search for what you need, and understand permissions, the rest becomes far easier to do.

You don’t need to memorise every command — you just need to recognise the patterns and know where to look when you’re not sure. 
Most of Linux is made up of small, predictable building blocks, and these are the ones you’ll use every day.
