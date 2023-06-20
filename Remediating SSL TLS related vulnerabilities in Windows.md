---
title: Remediating SSL/TLS related vulnerabilities on Windows Servers
tags: [security, windows]
created: 2021-06-21T00:43:47.773Z
modified: 2021-06-24T01:22:49.580Z
---

One day our regular Tenable scans revealed the following vulnerabilities in the local network:

- *[SSL Medium Strength Cipher Suites Supported (SWEET32)](https://www.tenable.com/plugins/nessus/42873)* (CVE-2016-2183)

- *[SSLv3 Padding Oracle On Downgraded Legacy Encryption Vulnerability (POODLE)](https://www.tenable.com/plugins/nessus/78479)* (CVE-2014-3566)

- *[SSL RC4 Cipher Suites Supported (Bar Mitzvah)](https://www.tenable.com/plugins/nessus/65821)* (CVE-2013-2566, CVE-2015-2808)

- [TLS Version 1.0 Protocol Detection](https://www.tenable.com/plugins/nessus/104743)

Here is the scanner's plugin output for the *SWEET32* vulnerability:

```
Medium Strength Ciphers (> 64-bit and < 112-bit key, or 3DES)

Name             Code             KEX    Auth    Encryption        MAC
--------------   ----------       ---    ----    --------------    ---
DES-CBC3-SHA     0x00, 0x0A       RSA    RSA     3DES-CBC(168)     SHA1
```

Manual scanning of the affected machines confirmed: *Triple DES* is advertised for client connections (on RDP port in this case):

```
root@MACHINE1:~# sslscan host1:3389 | grep CBC3
Accepted  TLSv1.2  112 bits  DES-CBC3-SHA
Accepted  TLSv1.1  112 bits  DES-CBC3-SHA
Accepted  TLSv1.0  112 bits  DES-CBC3-SHA
```

```
C:\programs\nmap>nmap --script ssl-enum-ciphers -p 3389 host1 | grep 3DES
|       TLS_RSA_WITH_3DES_EDE_CBC_SHA (rsa 2048) - C
|       TLS_RSA_WITH_3DES_EDE_CBC_SHA (rsa 2048) - C
|       TLS_RSA_WITH_3DES_EDE_CBC_SHA (rsa 2048) - C
```

Luckily all our vulnerabilities were detected for native Windows services like SMB, IIS, RDP, etc.
Native Windows services use [SCHANNEL](https://docs.microsoft.com/en-us/windows/win32/secauthn/secure-channel)
which means it is possible to fix them all by reconfiguring the SCHANNEL regisrty settings on the machines and rebooting them.

```
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers\Triple DES 168]
"Enabled"=dword:00000000
```

Had we had any third-party applications running on the affected ports (Apache webserver for example),
the fix would be different and application specific.

SCHANNEL contains a set of security methods:

- *Protocols* like PCT, SSL and TLS
- *Key exchange methods* like ECDHE, DHE and RSA
- *Cipher suites* like AES, MD5, RC4 and 3DES

They can be enabled or disabled
([1](https://docs.microsoft.com/en-US/troubleshoot/windows-server/windows-security/restrict-cryptographic-algorithms-protocols-schannel),
[2](https://techcommunity.microsoft.com/t5/core-infrastructure-and-security/demystifying-schannel/ba-p/259233))
in Windows Schannel by editing the registry.

Different ([1](https://docs.microsoft.com/en-gb/windows/win32/secauthn/cipher-suites-in-schannel),
[2](https://docs.microsoft.com/en-us/windows/win32/secauthn/protocols-in-tls-ssl--schannel-ssp-))
TLS cipher suites are enabled/disabled in Windows depending on its version.
New protocols get enabled, old ones become disabled by default and eventually not supported.
So instead of explicitly providing a list of allowed suites, it is better to disable the ones you don't need.

To maintain our company's SLAs, we couldn't just disable older protocols and cipher suites on production servers.
We had to be sure that our company's client applications will be able to keep establishing connections to the remediated servers.

So let's enable *verbose*
[SCHANNEL monitoring](https://docs.microsoft.com/en-us/troubleshoot/iis/enable-schannel-event-logging),
and wait for some data to be collected and review the logs. We can query the logs we need using Powershell: 

```
# Search for ID 36880 events for the past 30 days
$starttime = (Get-Date).AddDays(-30)
$filter = @{logname = 'System'; Id = 36880; StartTime = $starttime }
$events = Get-WinEvent $filter
$connections = @()

foreach ($event in $events) {
    $xmldata = [xml]$event.ToXml()
    $connection = [pscustomobject][ordered]@{
        Timestamp   = $event.TimeCreated
        Type        = $xmldata.Event.UserData.EventXML.Type
        Protocol    = $xmldata.Event.UserData.EventXML.Protocol
        CipherSuite = $xmldata.Event.UserData.EventXML.CipherSuite
        TargetName  = $xmldata.Event.UserData.EventXML.TargetName
    }
    $connections += $connection
}

$connections | Format-Table
```

```
Timestamp              Type   Protocol CipherSuite TargetName
---------              ----   -------- ----------- ----------
24/06/2021 12:59:02 AM client TLS 1.2  0xc030      client1.mycompany.corp
24/06/2021 12:54:11 AM client TLS 1.2  0xc030      client2.mycompany.corp
24/06/2021 12:34:06 AM client TLS 1.2  0xc030      client2.mycompany.corp
...
```

OK, our applications make TLS 1.2 connections only (no TLS 1.1 or TLS 1.0). It is time to disable older protocols in SCHANNEL.

It is possible to disable/enable protocols in SCHANNEL with an [IISCrypto](https://www.nartac.com/Products/IISCrypto) utility:

- Install it on an affected machine and launch
- Go to the *"Templates"* tab and select a *"PCI 3.2"* template

It will preselect quite a few older ciphers and protocols to be disabled: *DES, RC2, RC4, PCT, SSL and TLS 1.0/1.1*. It will not preselect *Triple DES 168* however so you have to do it manually:

- Go to the *"Schannel"* tab and untick *Triple DES 168*
- Go to the *"Advanced"* tab and click *Backup Registry*
- Click *Apply* to apply the new settings
- Restart the machine

However we had too many servers to use IISCrypto.
So [these](@attachment/CVE-2016-2183.reg.txt) registry settings can be applied to each machine by using (Powershell) scripts,
Group Policy Preferences or whatever automation tool is used in your organisation.
Another advantage of this method is that it doesn't explicitly enable *AES* cipher suites and *TLS 1.2* protocol.
*TLS 1.2* is [enabled](https://docs.microsoft.com/en-us/windows/win32/secauthn/protocols-in-tls-ssl--schannel-ssp-)
starting from Windows 8/Windows Server 2012 by default anyway.
You should have all olders versions
[decommissioned](https://docs.microsoft.com/en-us/troubleshoot/windows-server/windows-server-eos-faq/end-of-support-windows-server-2008-2008r2) already.

Registry settings had been applied to the affected machines and they had been restarted during their maintenance window.
Subsequent scanning confirmed that *Triple DES* is not advertised for client connections any longer:

```
root@MACHINE1:~# sslscan host1:3389 | grep CBC3 | wc -l
0
```
