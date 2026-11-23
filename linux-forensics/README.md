# Linux Forensics CTF Writeup

**Event:** MSU Cyber Club – Forensics meeting (Nov 23, 2026)

**Platform:** CTFd + Apache Guacamole (Ubuntu 24.04 lab box)

**Result:** 5/5 challenges solved (1,300 pts)

> Flags are redacted so this writeup doesn't spoil the challenges for future club members.

## Overview

The lab gave each participant a compromised Ubuntu server. The goal was to investigate it like an incident responder: find what the "attacker" left behind in logs, scheduled tasks, shell history and network config. Each artifact contained a flag in the format `dfctf{...}`.

## Skills practiced

- Navigating the Linux filesystem (`ls`, `cd`, `cat`)
- Using `sudo` to read protected files
- Knowing where Linux keeps evidence (`/var/log`, `/etc`, home directories)
- Searching for indicators with `grep`
- Recognizing what's normal versus suspicious on a server

---

## Challenge 1 – Web Server Logs (200)

**Prompt:** A suspicious request was made to the web server yesterday. Find what they were looking for.

**Approach:**
1. Identified the web server type by listing the log directory:
   ```bash
   ls /var/log          # found apache2/
   ```
2. The folder needed root access, so I listed it with sudo:
   ```bash
   sudo ls -la /var/log/apache2
   ```
   `access.log` was empty (0 bytes), and the older rotated log `access.log.1` had the data.
3. Read the rotated log:
   ```bash
   sudo cat /var/log/apache2/access.log.1
   ```

**Finding:** Dozens of normal `GET /` requests from localhost (`::1`), plus one request from an outside IP (`192.168.1.105`) for an unusual path that returned a 404. The requested path was the flag.

**Lesson:** Log rotation moves older entries into `.1`, `.2.gz` and so on. Always check the rotated files. Also, a single odd request among repetitive traffic is the classic sign.

**Gotcha:** `sudo cat /var/log/apache2/access.log*` failed because my normal user's shell expands the `*` before `sudo` runs, and that user can't read the directory. Fix: `sudo sh -c 'cat /var/log/apache2/access.log*'`.

---

## Challenge 2 – FTP Login Logs (200)

**Approach:**
```bash
sudo cat /var/log/vsftpd.log /var/log/vsftpd.log.1
```

**Finding:** Several `FAIL LOGIN` entries. One came from a different IP (`192.168.1.113`), and the attacker's username field contained the flag.

**Lesson:** vsftpd logs record connection attempts with the username tried. Failed logins from unknown IPs are an indicator of brute-force or probing activity.

---

## Challenge 3 – Scheduled Task (300)

**Approach:**
```bash
cat /etc/crontab
ls -la /etc/cron.d /etc/cron.hourly /etc/cron.daily
sudo crontab -l
```

**Finding:** `/etc/cron.d/` had one non-default file, `system-clean-up-time`, created on the same date as the other planted artifacts. Its name was made to look harmless.
```bash
cat /etc/cron.d/system-clean-up-time
```
The flag was hidden in a comment labeled "Auth token."

**Lesson:** Attackers use cron for persistence and give jobs innocent-sounding names. Compare file dates and look for anything that isn't a standard package file.

---

## Challenge 4 – User Command History (300)

**Approach:**
```bash
ls -la /home                    # found users: caj514, ethan
sudo sh -c 'cat /home/*/.bash_history /root/.bash_history'
```

**Finding:** User `ethan`'s history showed him running `./special_weapon.sh`, echoing the flag, then deleting the script with `rm`. He removed the file but not the record of running it.

**Lesson:** `.bash_history` is a hidden file (starts with a dot, so use `ls -la`) and often survives cleanup. It reconstructs exactly what a user did.

---

## Challenge 5 – Network Configuration (300)

**Approach:**
```bash
sudo grep -r dfctf /etc/hosts /etc/netplan /etc/network /etc/resolv.conf /etc/hostname 2>/dev/null
```

**Finding:** `/etc/hosts` had an added entry mapping `c2-server.local` to `10.10.1.42`, with the flag in a "beacon token" comment.

**Lesson:** Editing `/etc/hosts` lets malware reach a command-and-control (C2) server by name. Any unfamiliar entry there is worth investigating.

---

## Command cheat sheet

| Command | Purpose |
|---|---|
| `ls -la <dir>` | List all files, including hidden ones, with permissions and dates |
| `cat <file>` | Print a file |
| `sudo <cmd>` | Run as root (needed for most logs) |
| `sudo sh -c '<cmd with *>'` | Make wildcards expand as root |
| `grep -r <text> <dirs>` | Search files recursively for a string |
| `2>/dev/null` | Hide "permission denied" and other errors |
| Tab key | Autocomplete file names and avoid typos |

## Where to look on a Linux box

| Evidence | Location |
|---|---|
| Web requests | `/var/log/apache2/` or `/var/log/nginx/` |
| FTP logins | `/var/log/vsftpd.log` |
| SSH/sudo auth | `/var/log/auth.log` |
| Scheduled tasks | `/etc/crontab`, `/etc/cron.d/`, `crontab -l` |
| User activity | `/home/<user>/.bash_history` |
| Name resolution | `/etc/hosts`, `/etc/resolv.conf` |
| Network config | `/etc/netplan/` |
