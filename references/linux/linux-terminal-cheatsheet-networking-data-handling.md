# Linux Terminal Cheatsheet — Networking & Data Handling

This section covers the tools and commands you use when diagnosing networks, checking connectivity, inspecting ports, handling remote access, chaining commands, and working with text.

---

## 1. Networking Basics (ip, ping, routing)

```
ip a                   # list network interfaces + IPs
ip r                   # show routing table
hostname -I            # show assigned IPs
ping <host>            # basic connectivity test
ping -c 4 <host>       # fixed number of pings
```

---

## 2. Hostname & DNS Tools

```
nslookup <hostname>         # DNS query
dig <hostname>              # detailed DNS output
host <hostname>             # simple lookup
getent hosts <hostname>     # system resolver lookup
```

---

## 3. Checking Open Ports & Listening Services

```
ss -tuln                    # list listening ports
ss -tulnp                   # include process names
sudo ss -lptn | grep <port> # check a specific port
curl -I http://localhost:80 # check local service (headers only)
```

---

## 4. Network Testing with curl & wget

```
curl http://example.com
curl -I http://example.com
curl -L http://example.com
wget <url>
```

---

## 5. SSH Essentials

```
ssh user@host                 # SSH connection
ssh -p <port> user@host       # custom port
scp <src> <user@host:/dest>   # upload file
scp user@host:/src <dest>     # download file
```

---

## 6. Secure Copying & Remote File Transfers

```
scp file user@host:/path/          # upload
scp user@host:/path/file .         # download
rsync -av <src> <dest>             # sync
rsync -av --delete <src> <dest>    # mirror
```

---

## 7. Routes & Connectivity

```
ip r                         # routing table
traceroute <host>            # trace route
tracepath <host>             # alternative tracing
```

---

## 8. Redirection & Pipes

```
command > file.txt           # overwrite
command >> file.txt          # append
command < file.txt           # input
command1 | command2          # pipe output
```

Examples:

```
ls -l | grep ".log"
journalctl -u ssh | less
curl -I site | grep HTTP
```

---

## 9. Text Processing Basics

```
cut -d: -f1 /etc/passwd      # extract field
sort file.txt                # sort lines
uniq file.txt                # remove duplicates
awk '{print $1}' file.txt    # first column
sed 's/old/new/' file.txt    # replace text
```

---

## 10. File Transfers with wget & curl

```
wget <url>
curl -O <url>
curl -o name <url>
```

---

## 11. Checking Network Services Locally

```
curl -I http://localhost:<port>
ss -tuln | grep <port>
sudo journalctl -u <service> -f
```

---

## 12. Quick Connectivity Validation

```
ip a
ip r
ping -c 4 <gateway>
nslookup <domain>
ss -tuln | grep <port>
```

---

## 13. Network Utility Round-Up

```
nc -zv host <port>         # port test
arp -a                     # ARP table
ip neigh                   # neighbour discovery
ethtool <interface>        # link info
```
---

## 14. Summary
Networking and data-handling tools let you see how your server is behaving beneath the surface.
Once you can check interfaces, confirm routing, test services, and filter output, troubleshooting becomes far less guesswork.

Don’t memorise everything in this section.
You just need to know where to look, what each tool is for, and which command helps you answer the specific question you’re asking:

  - Is the server on the network? → ip a
  - Does it know where to send traffic? → ip r  
  - Can it reach anything? → ping  
  - Is DNS resolving correctly? → nslookup / getent  
  - Is the service actually listening? → ss  
  - Is it responding properly? → curl -I  
  - Is the issue on the path? → traceroute / tracepath

With these tools in your pocket, troubleshooting becomes much easier to follow — and your homelab becomes a lot less mysterious.
