# Docker on Linux as a Service Host  
*Turning a clean Linux box into a dependable Docker appliance — without chaos.*

A Docker host isn’t just “Linux with containers.”  
It’s a **purpose-built appliance** that behaves the same way every day, even when you’ve forgotten what you did three months ago. 
This guide walks through preparing the host properly: disk layout, mounts, structure, installation, and the expectations for how this thing should run.

If you’ve ever wondered *“why is Docker using 12GB of space when I only deployed Hello World?”* — this guide is precisely why I do things the Hvelvigen way.

---

## 1. What You’re Building

By the end of this, the machine will be:

- a **dedicated** Docker service host  
- cleanly structured under `/srv`  
- backed by a separate data disk (`sdb`)  
- delightfully predicatble and simple to manage and maintain 

We leave all application-specific messiness for Service Patterns.  
Here, we build the **platform**.

---

## 2. Before You Touch Docker: Prepare the Host

Before we install anything, we need to prepare the foundation the Docker host will stand on.  
This isn’t just “create a folder and hope for the best” — this is the part that stops your system becoming one of those mysterious machines where *nobody* knows where the data lives or why the OS disk is always 95% full.

A clean Docker host starts with a clean storage layout.  
And in Hvelvigen, that means one very simple rule:

**The OS lives on `sda`. Docker lives on `sdb`. They do not mix.**

Why?  
Because separating system and service data makes your host predictable, repairable, and boring — in the *good* way.  
When something goes wrong later (and it will, because homelabs are fun like that), you will know exactly where Docker keeps its runtime and exactly what is safe to wipe, move, or back up.

So, let’s prepare `sdb` properly.

---

### 2.1 Spot the disk  

The first step is confirming that the second disk actually exists and is visible to the OS.

```bash
lsblk -f
```

You’re looking for something like:

```
sda   (OS disk)
├─sda1 vfat     FAT32       5F73-BB3E                                 1G     1% /boot/efi
└─sda2 ext4     1.0         91fa959a-de5b-4a00-8f73-4a7ffa27fca7   21.6G    24% /
sdb   (the empty one)
```

`sha` = system;  
`sdb` = your shiny new home for Docker.

If you don’t see `sdb`, go add the virtual hard disk and try again.

---

### 2.2 Partition and format `sdb`

Docker creates a *lot* of small files, layers, overlay data, and container runtimes.  
Ext4 handles this extremely well, and sticking to one simple partition keeps things ... well, simple.

```bash
sudo parted /dev/sdb mklabel gpt
sudo parted -a opt /dev/sdb mkpart primary ext4 0% 100%
sudo mkfs.ext4 -L docker-data /dev/sdb1
```

Why GPT?  
Because it behaves consistently, supports modern tooling, and avoids the… “personality quirks” of MBR.

---

### 2.3 Create the mount point structure  

Hvelvigen has a standardised `/srv` layout so every Host Service Layer — Docker, HAProxy, Home Assistant, whatever — behaves the same way across machines.
Feel free to create your own structure, the key is to keep it consistent everywhere (and forgive yourself for that one place you didn't).

This consistency helps you (or someone else) understand the system instantly, even years later.

The host-level layout is:

```
/srv
  host-docker   # Docker’s own runtime and engine data
  docker        # Service Pattern data (one directory per service)
  logs          # Centralised logs for all services
```

Create them:

```bash
sudo mkdir -p /srv/host-docker
sudo mkdir -p /srv/docker
sudo mkdir -p /srv/logs
```

These directories will become the backbone of every service you deploy later.  
When someone asks “Where does *this* container store its data?”, the answer will *always* be under `/srv/docker/<service>` — no guessing.

---

### 2.4 Add sdb to fstab  

Now we mount the disk permanently so the host remains consistent across reboots.

First, grab the UUID (a stable identifier that won’t change if the disk order does):

```bash
lsblk -f
```

Then edit `fstab`:

```bash
cp /etc/fstab /etc/fstab.bak
sudo nano /etc/fstab
```

Add:

```
/dev/disk/by-uuid/<your-uuid-here>  /srv/host-docker  ext4  defaults  0  2
```

Why only `/srv/host-docker`?  
Because this is where Docker’s *runtime* lives — the part that grows unpredictably.  
Service Patterns (your long-term data) sit inside `/srv/docker/<service>` so they stay organised, portable, and easy to migrate.

Now mount it:

```bash
sudo mount -a
systemctl daemon-reload
df -h | grep srv
```

Expected result:

- `/srv/host-docker` is mounted  
- Nothing crashed  
- You didn’t accidentally fat-finger `fstab` into oblivion  

If something *did* explode, that’s exactly why we test with `mount -a` *before* rebooting, check `fstab` or restore your backup and try again.

---

### Why this matters (the educational bit)

A Docker host will generate:

- container layers  
- overlay mounts  
- volumes  
- logs  
- build caches  
- temporary metadata  

If all of this lands on the OS disk (`/var/lib/docker` by default), you will *absolutely* end up with a server that becomes unusable at the worst possible time.

By isolating Docker onto `sdb`:

- your OS stays clean  
- you always know where Docker data lives  
- disaster recovery becomes dramatically easier  
- performance improves due to fewer competing I/O workloads  
- backups become consistent and predictable  
- every host in your environment behaves the same way  

This is the kind of structure that stops small problems turning into big ones — and turns your homelab into something you can rely on confidently.

---

## 3. Installing Docker — The Sensible Way

Now that the storage is set up and the host has a clean foundation, we can actually install Docker.  
And here’s the important bit: **we install Docker in a way that keeps the host predictable.**

There are a thousand blog posts showing how to “get Docker running quickly.”  
Fast isn’t the goal here — **consistent, supportable, and non-chaotic** is the goal.

That’s why Hvelvigen uses the **official Docker repository** instead of whatever version your distro bundled three years ago.  
The official repo gives you reliable updates, consistent behaviour, and avoids mismatched tooling. Plus it's quick (network speed dependent).

### 3.0 Install prerequisites

```bash
sudo apt update
sudo apt install ca-certificates curl gnupg
```

### 3.1 Add the official Docker repository

We add the GPG key and repository in a clean, explicit way — no mysterious curl pipes or silent scripts.

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

Now tell apt where the Docker repo lives:

```bash
echo \
"deb [arch=$(dpkg --print-architecture) \
signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release; echo "$VERSION_CODENAME") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
```

Update the package list:

```bash
sudo apt update
```

### 3.2 Install the Docker engine and tools

This gives you:

- Docker Engine  
- Docker CLI  
- containerd  
- Buildx  
- Docker Compose v2  

All the tools you actually need for a modern Docker host.

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 3.3 Enable and start the Docker service

```bash
sudo systemctl enable --now docker
```

If the service starts cleanly, you’ve just built a properly configured, repo-backed Docker installation.

Check it with:

```bash
sudo docker info
```

You’ll get a wall of text — which is fine.  
We’ll validate the important bits later once we repoint Docker’s runtime to `/srv/host-docker`.

---

## Why all of this matters

Docker has a habit of becoming invisible once installed — right up until it isn’t.  
Installing it cleanly ensures:

- **repeatability** — every Docker host across your environment behaves the same  
- **supportability** — updates come from a predictable source  
- **sanity** — no “why is this host running Docker 19.x?” surprises  
- **compatibility** — Buildx, Compose v2, and containerd all stay aligned  

This is the “boring but essential” part of building a stable Docker appliance.  
Once Docker is installed cleanly, the fun parts (configuring its behaviour and deploying services) become smooth and predictable.

---

## 4. Configure Docker to Behave  
Installing Docker is only half the job.  

No one wants Docker filling `/var/lib/docker` like an unsupervised toddler.
Left untouched, Docker will happily store its runtime, layers, and volumes in `/var/lib/docker` — which is exactly what we’re avoiding.

A **Hvelvigen Docker Host** must:

- keep the OS disk clean  
- store all Docker engine data on `/srv/host-docker`  
- behave predictably across reboots  
- make troubleshooting simple rather than “archaeological”  

So now we teach Docker where it should live, how it should log, and how to confirm it’s actually doing what we asked.

---

## 4.1 Create the Docker daemon configuration

Docker uses `/etc/docker/daemon.json` to define how the engine behaves.  
If this file doesn’t exist, Docker falls back to defaults — which is exactly what fills your OS disk.

Create the config directory:

```bash
sudo mkdir -p /etc/docker
```

Open the daemon config:

```bash
sudo nano /etc/docker/daemon.json
```

Add the following:

```json
{
  "data-root": "/srv/host-docker/runtime",
  "log-driver": "json-file",
  "log-level": "warn",
  "storage-driver": "overlay2"
}
```

### Why these settings?

- **data-root** → relocates all Docker engine data (layers, containers, volumes) onto `sdb`  
- **log-driver: json-file** → predictable, easy to inspect, no daemon-side surprises  
- **log-level: warn** → reduces noise; informational logs are handled per-service, not globally  
- **overlay2** → stable, fast, modern storage driver  

This configuration ensures Docker behaves like an appliance rather than a wild animal.

---

## 4.2 Restart Docker to apply changes

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

Docker now *should* be using `/srv/host-docker/runtime`  
Let’s confirm that:

```bash
docker info | grep "Docker Root Dir"
```

Expected:

```
Docker Root Dir: /srv/host-docker/runtime
```

If it still says `/var/lib/docker`, something earlier wasn’t configured correctly — fix that before moving on.

---

## 4.3 Validate the Docker Installation (Hello World Test)

Before deploying anything real, we test the engine, networking, permissions, and storage all in one simple command.

```bash
docker run --rm hello-world
```

This confirms:

- the Docker engine works  
- networking is functional  
- the user can run containers without sudo  
- images pull correctly  
- the runtime directory is writable  

If the command errors, revisit:

- **3.1** (docker group)  
- **4.1** (daemon.json)  

### Check where Docker actually wrote data

Now we look at the locations Docker uses behind the scenes:

```bash
sudo du -sh /var/lib/docker 2>/dev/null
sudo du -sh /srv/host-docker/runtime
```

Expected outcome:

- `/var/lib/docker` → **empty or very small**  
- `/srv/host-docker/runtime` → **contains actual data**  

If the reverse is true, Docker is not obeying the configuration — correct this before continuing.

### Clean up the test container/image

We only used hello-world to verify the host.  
It doesn’t need to stay around.

```bash
docker image rm hello-world
```

This returns your pristine Docker Host to a clean slate, ready for real workloads.

---

## Why this configuration matters

By explicitly defining where Docker stores its data:

- the OS disk stays clean and predictable  
- Docker’s behaviour is transparent  
- hosts remain consistent across your environment  
- you avoid a movie stopping halfway through an epic scene because `/` became full on the host! 

This is the point where the host stops being “just Linux with Docker installed” and becomes a **Docker appliance** — reliable, repeatable, and fully aligned with the Hvelvigen (or your own) service model.

---

## 5. The Host-Level Rules  
These rules are what separate a clean, predictable Docker appliance from the kind of host that slowly accumulates weird containers, stray ports, and logs that nobody asked for.  
They exist to keep the system tidy, debuggable, and safe — and to make future-you genuinely grateful you followed them.

---

### Rule 1: Nothing runs directly on the host  

If you need a quick web server running *on the host*, you’re either: 

- testing something naughty
- or you forgot Docker exists

If you're tempted to run a quick Python server or spin up NGINX directly on the OS, stop.  
That’s how machines turn into unmaintainable soup.

The host has **one job**:  
**run Docker reliably**.

If you need something:

- package it into a container  
- or use a Service Pattern  
- or delete it because you were just curious  

Running workloads directly on the host bypasses logs, structure, security expectations, and undoability.  
It creates “mystery behaviour” — the hardest kind to debug.

---

### Rule 2: One network layer per service  

Networking is where containers love to cause chaos if left unchecked.

We do **not** expose ports casually.  
We do **not** dump everything onto the host network.  
We do **not** let containers fight over the same addresses.

Instead:

- Each Service Pattern gets its **own network**  
- Only explicitly required ports are published  
- LAN-bridging happens **only** when there’s a compelling reason  
- Internal-only services stay internal-only  

This keeps your environment sane, controlled, and debuggable.  
If something talks to something it shouldn’t, it’s because *you said it could* — not because Docker guessed.

---

### Rule 3: No pet containers  

A “pet container” is something you spun up to test a thing, explore a CLI, or poke at an idea…  
and then forgot about.

You know the type:

```bash
docker run -it ubuntu bash
```

Then you get distracted, walk away, and three months later you can’t remember what it was for or why it’s still running.

Pet containers become:

- random CPU spikes  
- stale network bindings  
- old volumes consuming space  
- “why does this container exist?” moments  

If you started a container for experimentation, delete it when you're done.  
It's useful but not necessary to track the lifecycle of your containers to avoid this very issue.

---

### Rule 4: Logs live in `/srv/logs`  

Every service writes logs somewhere.  
If you let Docker choose that “somewhere”, you’ll end up with:

- logs in container layers  
- logs under `/var/lib/docker`  
- logs inside bind mounts you didn’t know existed  
- logs you can’t find during an outage  

Hvelvigen follows a simple rule:

```
/srv/logs/<service>/
```

This gives you:

- consistent log paths across every host  
- readable, rotated logs that don’t fill disks  
- sane monitoring and troubleshooting  
- a predictable backup story  

The host becomes *boringly consistent* — which is exactly what you want.

---

### Why these rules matter

Rules 1–4 together create a Docker host that is:

- stable  
- transparent  
- easy to audit  
- easy to recover  
- predictable across upgrades  
- identical to every other Docker host you deploy  

These constraints may feel opinionated, but they give you the kind of reliability normally reserved for expensive commercial appliances — except you understand exactly how this one works.

---

## 6. Updates & Maintenance  
Unlike some systems (Hi Windows! Oh, you'll be back in a bit, OK!), Linux doesn’t punish you for updating.

This section explains **how to keep the host healthy long-term**, and more importantly, *why* we update the way we do.

---

### System updates  
The OS should stay patched and secure. A clean, minimal server means updates are usually quick and drama-free.

```bash
sudo apt update && sudo apt upgrade
```

Why keep the OS updated?

- container runtimes depend on kernel features  
- outdated SSL libraries cause strange TLS failures  
- security patches matter even in homelabs  
- consistency across hosts reduces debugging time  

A Docker host with an ancient kernel is a ticking time-bomb disguised as a server.

---

### Docker updates  
Docker updates arrive through the same mechanism — the official Docker repository you added earlier.

No weird scripts.  
No curl-to-bash surprises.  
No version mismatches between Docker, containerd, or Buildx.

You update the OS, and Docker comes along for the ride.  
Simple and **predictable**.

---

### Pruning  
Pruning is “delete everything Docker thinks you don’t need anymore”.  
Which is why we don’t automate it.

Automatic prune jobs often lead to:

- vanished containers  
- missing volumes  
- broken builds  
- confused future-you  
- the classic “I swear this worked yesterday…” moment  

Instead, pruning is a **deliberate, conscious action**:

```bash
docker image prune
docker volume prune
docker container prune
```

Use it when you know what you’re deleting, not because a cron job decided you’re done with it.

If you ever do need to reclaim space safely, Hvelvigen encourages following a simple rule:

**If you can't explain what you’re deleting, don’t delete it.**

---

### Why updates & maintenance matter  
A Docker host is a long-lived appliance.  
Keeping it healthy means:

- fewer outages  
- fewer rebuilds  
- predictable behaviour during deployments  
- predictable behaviour during rollbacks  
- fewer “mystery failures” caused by outdated libraries  
- confidence that the host will behave the same next month as it does today  

Maintenance isn’t glamorous — but it’s what keeps your homelab smooth, stable, and under control.
Or, think of it another way. If you're not spending the time fixing things, you can use it to build things.

---

## 7. Security Expectations  
A Docker Host isn’t just “a server that runs containers” — it’s an appliance you trust to run multiple services safely.  
Security here isn’t about paranoia or enterprise checklists; it’s about **not giving future-you unnecessary heart attacks**.

These expectations form the minimum baseline for a Hvelvigen-compliant host.  
They’re simple, they’re practical, and they stop 99% of the daft problems **I** accidentally created.

---

### 🔒 The Docker API socket is never exposed  
Ever.

Docker exposes a powerful API at `/var/run/docker.sock`, and giving remote access to it is effectively handing out root permissions wrapped in JSON.

If a guide online tells you to expose it over TCP “for convenience”, close the tab.

---

### 🚫 No privileged containers unless absolutely necessary  
Privileged mode turns a container into a “VM with superpowers”.  
It grants access to host kernel features, devices, and namespaces.

Most workloads do **not** need this.  
If a container claims it does, double-check the documentation — twice.

Privileged containers are:

- harder to secure  
- easier to break  
- capable of modifying the host in unintended ways  

Treat them like power tools: only use them if you genuinely understand the risk.

---

### 🔥 UFW (or your preferred firewall) stays enabled  
A Docker host should not behave like a friendly neighbourhood open port festival (Hello Sailor!).

Enable and configure UFW:

```bash
sudo ufw allow ssh
sudo ufw enable
```

Then allow only what the **Service Patterns** require.  
No extra ports.  
No “just in case” gaps.

Security is at its best when it’s boring.

---

### 🔑 SSH is locked down  
SSH is your lifeline into the host — protect it.

Expectations:

- no password authentication  
- key-based auth only  
- non-default port optional but acceptable  
- fail2ban or equivalent is a nice extra  
- root login disabled unless you’re deliberately doing something spicy  

If SSH is tight, the entire host benefits.

---

### 🤫 Secrets never live in the Host Service Layer  
The Docker Host is *not* where you put API keys, credential files, or environment secrets.

Those belong in:

- service-specific config folders  
- Docker Compose secrets  
- Vault / external secret management (if you're going fancy)  

This keeps the host clean and reduces the blast radius if anything leaks.

---

### 👥 The docker group is for admins only  
The `docker` group can effectively escalate to root.  
So only add:

- your admin account  
- automation accounts (if you use them)  

No service accounts, no random logins, no convenience users.

---

### Why these expectations matter  
Security here isn’t about fear — it’s about **maintainability**.

A secure Docker host is:

- harder to accidentally break  
- easier to audit  
- safer to expose on networks  
- predictable for every Service Pattern  
- consistent with every other host in your environment  

Security by default = fewer problems later.

A Docker host configured this way won’t surprise you, won’t leak data, and won’t open weird ports because someone followed a tutorial from 2017.

---

## 8. Validation Checklist  
Your host is considered “Hvelvigen-compliant” when:

- [ ] `/srv/host-docker` is mounted on sdb  
- [ ] `/srv/docker` exists for Pattern data  
- [ ] `/srv/logs` exists  
- [ ] Docker root dir is `/srv/host-docker/runtime`  
- [ ] No containers live outside Service Patterns  
- [ ] The host boots cleanly without yelling in the logs  
- [ ] You can explain its structure to someone else without cringing  

If all of those are true, congratulations — you’ve built a solid Docker host.

---

## 9. Summary  
This guide exists so Docker doesn’t eat your OS disk, your sanity, or your weekend.  
With the host prepared properly — dedicated disk, defined structure, clean configuration — everything above it becomes easier to deploy, maintain, and migrate.

This is the foundation your Service Patterns will stand on.  
And (bias aside) it’s a good one.
