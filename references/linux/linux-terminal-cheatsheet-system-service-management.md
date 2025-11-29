# Linux Terminal Cheatsheet — System & Service Management

This section covers the commands you’ll use once you’re past the basics and into the day‑to‑day running of an Ubuntu Server: managing software, inspecting services, checking logs, understanding system health, and keeping everything behaving predictably.

It assumes you can already move around the filesystem and edit files.  
Now we look at what keeps the server running.

---

## 1. Package Management (APT)

APT handles installing, upgrading, and removing software.

```
sudo apt-get update                 # refresh package lists
sudo apt-get upgrade -y             # install available updates
sudo apt-get install <package>      # install a package
sudo apt-get remove <package>       # remove a package
sudo apt-get autoremove             # clean unneeded dependencies
apt-cache search <term>             # search for packages
dpkg -l | grep <term>               # list installed packages
```

---

## 2. Services & Daemons (systemctl)

Systemd manages background processes (“services”).

```
sudo systemctl status <service>     # check status
sudo systemctl start <service>      # start
sudo systemctl stop <service>       # stop
sudo systemctl restart <service>    # restart after changes
sudo systemctl enable <service>     # start automatically at boot
sudo systemctl disable <service>    # disable autostart
sudo systemctl reload <service>     # reload config (if supported)
```

---

## 3. System Logs & Diagnostics

Logs explain what happened and why.

```
journalctl -u <service>             # logs for a service
journalctl -u <service> -f          # follow logs live
journalctl -xe                      # recent system errors
tail -f /var/log/syslog             # general system logging
dmesg                               # kernel messages
```

---

## 4. Processes & Resource Monitoring

See what your server is busy with.

```
top                      # live process list
btop                     # cleaner, readable monitor
ps aux                   # detailed process snapshot
free -h                  # memory usage
uptime                   # load and uptime
df -h                    # disk usage
du -sh <path>            # directory size summary
```

---

## 5. Disk & Filesystem Usage

Storage issues often cause unexpected failures.

```
df -h                        # mounted disks and usage
du -sh <folder>              # folder size summary
lsblk                        # list block devices
mount                        # display mounts
sudo mount <dev> <path>      # mount manually
sudo umount <path>           # unmount
```

---

## 6. Working with Archives

```
tar -czvf archive.tar.gz <folder>   # create archive
tar -xzvf archive.tar.gz            # extract archive
zip file.zip file1 file2            # create zip
unzip file.zip                      # extract zip
```

---

## 7. Scheduling Tasks (cron)

Automate tasks like backups and scripts.

```
crontab -e                   # edit user cron
crontab -l                   # list cron
sudo crontab -e              # root cron
```

Cron format:

```
* * * * *  command
| | | | |
| | | | └── day of week
| | | └──── month
| | └────── day of month
| └──────── hour
└────────── minute
```

Example: run at 3:30 daily

```
30 3 * * * /usr/local/bin/backup.sh
```

---

## 8. System Identity & Info

```
hostnamectl                 # hostname + OS info
uname -a                    # kernel info
lsb_release -a              # Ubuntu version
whoami                      # current user
uptime                      # load + uptime
cat /etc/os-release         # OS metadata
```

---

## 9. Package & Service Troubleshooting

```
systemctl status <service>          # check status
journalctl -u <service> -xe         # service logs
sudo nano /etc/<service>/<file>     # inspect config
ss -tuln | grep <port>              # check listening port
sudo systemctl restart <service>    # restart
```

---

## 10. Firewall (UFW)

```
sudo ufw status
sudo ufw allow <port>
sudo ufw deny <port>
sudo ufw delete allow <port>
sudo ufw enable
```

Document firewall changes when you make them — future‑you will appreciate it.
