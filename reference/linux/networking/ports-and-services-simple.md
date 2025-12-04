# Ports & Services — The Simple Version
A clear, beginner-friendly explanation of what ports are, how services use them, and why “it works here but not there” could very well be a port problem.

---

## 1. The Core Idea
An IP address identifies **the device**.  
A port identifies **the specific service** on that device.

Think of a server as a block of flats (apartments):
- same building (IP)  
- different doors (ports)  

Common examples:
```
22   SSH
80   HTTP
443  HTTPS
8080 Dashboards
3000 Admin tools
```

If a service isn’t listening on the port you expect, nothing will reach it — no matter how angrily you refresh the page.

---

## 2. Docker Port Mapping (The Confusing Bit)
Docker introduces a second layer of ports:

```
-p 8080:80
```

Means:
- the host listens on **8080**  
- the container listens on **80**  
- you access it using:  
  ```
  http://host-ip:8080
  ```

A common beginner mistake is thinking you connect *to the container*.  
You don’t.  
You connect to the **host**, which forwards traffic to the container.

---

## 3. Listening Addresses (Why It Works Here but Not There)
A service can choose where to listen:

- `127.0.0.1:<port>` → only the local machine can access it  
- `<LAN IP>:<port>` → only that network interface  
- `0.0.0.0:<port>` → every interface on the device  

If something works on the device itself but not from your laptop, check the service isn't bound to **127.0.0.1** instead of the LAN IP.

I've had “Why can’t I reach it?” moments that were solved by checking the listening address.

---

## 4. Checking What’s Listening
To see what ports are open and which processes own them:

```
ss -tuln
ss -tulnp     # includes process names
```

If you don’t see the port listed, the service isn’t running  
**Note** Glaring at the terminal will not fix this. Snacc helps!

---

## 5. Common Port Mistakes
- Two services fighting for the same port  
- Containers bound to localhost only  
- Forgetting UFW is blocking traffic  
- Using the wrong port in the browser  

These account for an non-trivial portion of “the service is running but unreachable” incidents.

---

## 6. Summary
Ports are simple:

- IP = the device  
- Port = the service  
- Docker can remap them  
- Firewalls can block them  
- Listening address decides who can connect  

Master ports, and you'll be able to make the most out of multi-service servers.

