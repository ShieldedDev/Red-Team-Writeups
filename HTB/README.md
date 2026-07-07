# Hack The Box Assessments

This directory contains my collection of **Hack The Box** machine assessments documented as professional-style offensive security walkthroughs.

Rather than publishing simple "flag walkthroughs," the goal of these reports is to document each machine as if it were a real penetration testing engagement. Every assessment focuses on the complete attack lifecycle—from reconnaissance to post-exploitation—while explaining the reasoning behind each decision.

Each writeup emphasizes methodology over memorization and aims to demonstrate practical offensive security skills, systematic enumeration, vulnerability validation, exploitation, privilege escalation, and clear technical documentation.

---

# Assessment Methodology

Every machine follows a consistent methodology inspired by real-world penetration testing engagements.

```
Reconnaissance
      │
      ▼
Service Enumeration
      │
      ▼
Attack Surface Analysis
      │
      ▼
Vulnerability Research
      │
      ▼
Initial Access
      │
      ▼
Privilege Escalation
      │
      ▼
Post Exploitation
      │
      ▼
Lessons Learned
```

Each report documents not only **what** worked, but also **why** a particular approach was chosen and what observations influenced the next step.

---

# Report Structure

Each machine directory contains:

```
Machine/
│
├── README.md
├── <Machine>-Walkthrough.pdf
└── Images/
```

The reports generally include:

- Overview
- Attack Path
- Reconnaissance
- Enumeration
- Vulnerability Analysis
- Exploitation
- Privilege Escalation
- Findings
- Lessons Learned
- References

---

# Machines

| Machine | Platform | OS | Primary Focus |
|----------|----------|----|---------------|
| Blue | Windows | Windows 7 | MS17-010 (EternalBlue) |
| Connected | Linux | CentOS | FreePBX Vulnerability Chaining |
| Devel | Windows | Windows Server | IIS & WebDAV Exploitation |
| Grandpa | Windows | Windows Server 2003 | IIS 6.0 WebDAV & Local Privilege Escalation |
| Jerry | Windows | Windows Server | Apache Tomcat Manager |
| Lame | Linux | Ubuntu | Samba Username Map Script |
| Legacy | Windows | Windows XP | MS08-067 |
| Netmon | Windows | Windows Server | PRTG Network Monitor |
| Nibbles | Linux | Ubuntu | Nibbleblog RCE |
| Optimum | Windows | Windows Server | HttpFileServer (HFS) RCE |
| Reactor | Linux | Linux | Web Application Assessment |

---

# Skills Demonstrated

Across these assessments, the following techniques and concepts are covered.

## Reconnaissance

- Active Host Discovery
- Service Enumeration
- Banner Grabbing
- Version Detection
- HTTP Fingerprinting
- SMB Enumeration

## Web Application Security

- Directory Enumeration
- Virtual Host Discovery
- File Upload Abuse
- Authentication Bypass
- Web Shell Deployment
- Application Fingerprinting

## Network Security

- SMB Enumeration
- FTP Enumeration
- WebDAV Assessment
- HTTP/HTTPS Analysis
- SSH Enumeration

## Exploitation

- Public Exploit Validation
- Metasploit Framework
- Searchsploit
- Manual Exploit Adaptation
- CVE Validation
- Vulnerability Chaining

## Post Exploitation

- Shell Stabilization
- Interactive TTY Upgrades
- Meterpreter
- File Transfer
- User Enumeration
- Configuration Review

## Privilege Escalation

- Windows Token Impersonation
- Linux Misconfigurations
- Service Abuse
- Writable Files
- Automated Enumeration (LinPEAS)
- Manual Privilege Escalation

---

# Tools Used

The following tools are frequently used throughout these assessments.

- Nmap
- Metasploit Framework
- Searchsploit
- Netcat
- Burp Suite
- Gobuster
- LinPEAS
- WinPEAS
- Python
- FTP Client
- SMBClient
- CrackMapExec (where applicable)

---

# Objectives

The purpose of these reports is to:

- Demonstrate structured penetration testing methodology.
- Improve technical documentation skills.
- Practice realistic offensive security workflows.
- Build a portfolio of reproducible assessments.
- Record lessons learned during each engagement.

These walkthroughs are written to document the thought process behind each assessment rather than simply presenting exploit commands.

---

# Disclaimer

All assessments documented in this directory were performed exclusively against **Hack The Box** machines within an authorized training environment.

The material is provided for educational purposes and to demonstrate offensive security methodology, technical documentation, and penetration testing practices.