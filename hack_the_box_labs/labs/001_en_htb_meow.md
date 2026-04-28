# 📘 Hack The Box: Meow

> 🔗 **Open the lab:**  
> https://app.hackthebox.com/machines/Meow

---

## 📑 Table of Contents

- 🎯 Goal
- 🧠 Key Idea
- 🔍 Solution Walkthrough
- 💥 Vulnerability
- 🛡️ Recommendations
- 📋 Task Answers
- 🔐 Solution
- 🛠️ Checklist

---

## 🎯 Goal

Gain access to the target machine and retrieve the flag using basic enumeration and exploitation techniques.

---

## 🧠 Key Idea

The machine is vulnerable due to:

- exposed Telnet service  
- empty password for root  

This is a misconfiguration vulnerability.

---

## 🔍 Solution Walkthrough

### Step 1 — Connectivity check

```
ping <TARGET_IP>
```

---

### Step 2 — Port scanning

```
nmap -sV <TARGET_IP>
```

📌 Result:
- 23/tcp open  
- telnet service  

---

### Step 3 — Connect

```
telnet <TARGET_IP>
```

---

### Step 4 — Credentials

Try:
- admin  
- administrator  
- root  

📌 Success:
- root / empty password  

---

### Step 5 — Flag

```
ls
cat flag.txt
```

---

## 💥 Vulnerability

- Telnet (unencrypted)  
- root without password  
- weak authentication  

👉 Full compromise possible

---

## 🛡️ Recommendations

- Disable Telnet  
- Use SSH  
- Strong passwords  
- Disable root login  
- Security audits  
- Firewall  

---

## 📋 Task Answers

<details>
<summary>Show Task 1–7 answers</summary>

Task 1: Virtual Machine  
Task 2: terminal  
Task 3: openvpn  
Task 4: ping  
Task 5: nmap  
Task 6: telnet  
Task 7: root  

</details>

---

## 🔐 Solution

<details>
<summary>Show solution</summary>

root (no password)  
flag.txt → b40abdfe23665f766f9c61ecba8a4c19  

</details>

---

## 🛠️ Checklist

- [ ] VPN connection  
- [ ] Ping  
- [ ] Scan  
- [ ] Service found  
- [ ] Credentials  
- [ ] Access  
- [ ] Flag  
