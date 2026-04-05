# 📘 Hack The Box: Fawn

> 🔗 https://app.hackthebox.com/machines/Fawn

---

## 🎯 Goal

Access the FTP server and retrieve the flag file.

---

## 🧠 Key Idea

The server is misconfigured:
- anonymous login is enabled
- no authentication required

This allows anyone to access files.

---

## 🔍 Solution Walkthrough (detailed)

### Step 1 — Connectivity check

```
ping <TARGET_IP>
```

📌 We check if the machine is reachable.

---

### Step 2 — Port scanning

```
nmap -sV <TARGET_IP>
```

📌 We identify open ports and services.

Result:
- 21/tcp open
- ftp service
- vsftpd 3.0.3
- Unix OS

---

### Step 3 — Connect to FTP

```
ftp <TARGET_IP>
```

📌 We connect and get login prompt.

---

### Step 4 — Anonymous login

```
Username: anonymous
Password: (anything)
```

📌 This works due to misconfiguration.

---

### Step 5 — List files

```
ls
```

📌 We find:
```
flag.txt
```

---

### Step 6 — Download file

```
get flag.txt
```

📌 File is downloaded locally.

---

### Step 7 — Read flag

```
cat flag.txt
```

---

## 💥 Vulnerability

- anonymous access  
- no authentication  
- unencrypted FTP  

---

## 🛡️ Recommendations

- disable anonymous login  
- use SFTP  
- enforce authentication  
- restrict access  

---

## 📋 Task Answers

<details>
<summary>Show Task 1–11 answers</summary>

1. File Transfer Protocol  
2. 21  
3. SFTP  
4. ping  
5. vsftpd 3.0.3  
6. Unix  
7. ftp -?  
8. anonymous  
9. 230  
10. ls  
11. get  

</details>

---

## 🔐 Flag

<details>
<summary>Show</summary>

035db21c881520061c53e0536e44f815

</details>

---

## 🛠️ Checklist

- [ ] Connectivity check  
- [ ] Scan  
- [ ] FTP access  
- [ ] Anonymous login  
- [ ] File discovery  
- [ ] Download  
- [ ] Flag obtained  
