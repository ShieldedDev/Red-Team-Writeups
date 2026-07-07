# Hack The Box — Lame

> **Platform:** Hack The Box  
> **Machine:** Lame  
> **Difficulty:** Easy  
> **Operating System:** Linux  
> **Assessment Type:** Black-Box  
> **Objective:** Obtain user and root access through enumeration and exploitation of exposed services.

---

# Overview

Lame is one of Hack The Box's classic beginner machines, but it demonstrates an important lesson that applies to real-world engagements: **thorough enumeration often reveals vulnerable legacy software that can provide direct access to a target.**

The objective of this assessment was to identify externally exposed services, enumerate them, validate potential attack vectors, and obtain complete control over the target system.

Although the machine is considered easy, it reinforces an important penetration testing principle:

> Enumeration drives exploitation.

Throughout this assessment I documented not only what worked, but also why each decision was made before moving to the next phase.

---

# Attack Path

```
Reconnaissance
      │
      ▼
Service Enumeration
      │
      ▼
Identify Legacy Services
      │
      ▼
Validate Known Vulnerabilities
      │
      ▼
Anonymous FTP Enumeration
      │
      ▼
SMB Exploitation (CVE-2007-2447)
      │
      ▼
Interactive Shell
      │
      ▼
User Enumeration
      │
      ▼
Root Access
```

---

# Initial Reconnaissance

The first objective was to understand the exposed attack surface of the target.

I started with an Nmap scan using the default NSE scripts together with service version detection.

```bash
nmap -sC -sV -oN nmap_scan 10.129.166.254
```

## Scan Result

> 📷 **Screenshot**

```
(Images/nmap_scan.png)
```

The scan identified four publicly accessible services.

| Port | Service | Version |
|-------|----------|----------|
|21|FTP|vsFTPd 2.3.4|
|22|SSH|OpenSSH 4.7p1|
|139|NetBIOS|Samba|
|445|SMB|Samba 3.0.20|

At this point, two services immediately attracted attention.

- **vsFTPd 2.3.4**
- **Samba 3.0.20**

Both are older versions with publicly documented vulnerabilities, making them the primary candidates for further investigation.

---

# Vulnerability Research

Rather than immediately launching exploits, I first verified whether known public vulnerabilities existed for the discovered versions.

## Checking vsFTPd

```bash
searchsploit vsftpd 2.3.4
```

> 📷 **Screenshot**

```
Images/searchsploit-ftp.png
```

The results confirmed the existence of the well-known **vsFTPd 2.3.4 backdoor**.

However, that vulnerability affected only a maliciously distributed version of the software.

Since version numbers alone do not guarantee exploitability, I decided not to rely on this attack path without additional evidence.

---

## Checking Samba

Next, I investigated the Samba version.

```bash
searchsploit Samba 3.0.20
```

> 📷 **Screenshot**

```
Images/searchsploit-samba.png
```

Among the available exploits, one immediately stood out.

```
Username Map Script Command Execution
```

This vulnerability (CVE-2007-2447) allows remote command execution against vulnerable Samba installations when the username mapping script is enabled.

Because the detected version matched the vulnerable release, SMB became the primary attack vector.

---

# Additional SMB Enumeration

Before attempting exploitation, I performed another round of SMB enumeration using Nmap vulnerability scripts.

```bash
nmap --script smb-vuln* 10.129.166.254 -oN nmap_smb-vuln
```

> 📷 **Screenshot**

```
Images/nmap_smb-vuln.png
```

Interestingly, the NSE scripts did not explicitly identify CVE-2007-2447.

This is not uncommon.

Nmap vulnerability scripts detect only a subset of publicly known issues and should never be treated as definitive proof that a target is secure.

Since both the service version and public exploit database aligned, the exploitation path remained valid.

---

# FTP Enumeration

The Nmap scan also reported that anonymous FTP authentication was enabled.

I verified this manually.

```bash
ftp 10.129.166.254
```

> 📷 **Screenshot**

```
Images/ftp.png
```

Anonymous authentication succeeded without requiring credentials.

After inspecting the available directories and files, no sensitive information or useful artifacts were found.

Although this path did not lead to initial access, documenting unsuccessful attack paths is equally valuable because it demonstrates that the service was properly investigated before moving on.

---

# Initial Access

With Samba identified as the most promising target, I searched Metasploit for an appropriate exploit module.

```text
search samba 3.0.20
```

> 📷 **Screenshot**

```
Images/msf_search-samba.png
```

Metasploit included the **Username Map Script** exploit module.

After configuring the required parameters:

- RHOSTS
- LHOST
- LPORT

I executed the exploit.

> 📷 **Screenshot**

```
Images/msf_samba-exploited.png
```

The exploit successfully returned a command shell, confirming remote code execution against the target system.

Initial access had been achieved.

---

# Shell Stabilization

The initial shell was functional but limited.

To improve usability during post-exploitation, I upgraded it to a fully interactive TTY.

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

This enabled a much more stable shell with proper terminal behavior.

---

# User Enumeration

With an interactive shell available, I began exploring the filesystem.

```bash
cd /home

ls
```

Three user directories were present.

```
ftp
makis
service
user
```

The `makis` directory contained the user flag.

```bash
cat /home/makis/user.txt
```

> 📷 **Screenshot**

```
Images/lame_user-flag.png
```

Successfully obtaining the user flag confirmed complete user-level compromise.

---

# Root Access

One interesting aspect of this machine is that exploiting Samba through the Username Map Script vulnerability immediately provides execution as the **root** user.

As a result, no additional privilege escalation techniques were required.

Navigating to the root directory confirmed unrestricted access.

```bash
cd /root

ls

cat root.txt
```

> 📷 **Screenshot**

```
Images/lame_root-flag.png
```

Reading `root.txt` completed the objective of the assessment.

---

# Findings

| Finding | Severity |
|----------|----------|
|Legacy Samba installation vulnerable to CVE-2007-2447|Critical|
|Anonymous FTP Login Enabled|Low|
|Outdated Service Versions|High|

---

# Lessons Learned

This machine reinforces several practical lessons that apply well beyond Hack The Box.

- Enumeration is the most important phase of an assessment.
- Version detection frequently reveals publicly known attack vectors.
- Public exploit databases should be used to validate findings rather than blindly launching exploits.
- Nmap vulnerability scripts are useful but should never be considered exhaustive.
- Dead ends, such as anonymous FTP in this case, should still be documented because they demonstrate a systematic assessment methodology.
- Understanding *why* an exploit works is significantly more valuable than simply executing it.

---

# Tools Used

- Nmap
- Searchsploit
- Metasploit Framework
- FTP Client
- Python (TTY Upgrade)

---

# References

- CVE-2007-2447
- Samba Username Map Script Command Execution
- Exploit-DB
- Hack The Box — Lame

---

> **Disclaimer**
>
> This walkthrough documents an assessment performed exclusively within the authorized Hack The Box laboratory environment. It is intended for educational purposes and to demonstrate penetration testing methodology in a controlled setting.
