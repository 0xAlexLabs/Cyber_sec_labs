# Bandit Level 0

## 🔗 Level Link
https://overthewire.org/wargames/bandit/bandit0.html

---

## 🧠 Level Goal
Connect to the Bandit server using SSH.

---

## 📌 Given
- Host: bandit.labs.overthewire.org
- Port: 2220
- Username: bandit0
- Password: bandit0

---

## 💻 Solution

```bash
ssh bandit0@bandit.labs.overthewire.org -p 2220
```

---

## 📖 Explanation

Command:

```bash
ssh bandit0@bandit.labs.overthewire.org -p 2220
```

Breakdown:

- `ssh` — Secure Shell program used for remote connections  
- `bandit0@bandit.labs.overthewire.org` — specifies the user (`bandit0`) and the server address  
- `-p 2220` — connects to a non-standard port (default SSH port is 22)  

How it works:
- establishes a secure connection to the server  
- attempts login as user `bandit0`  
- prompts for a password  

On first connection:
- confirm authenticity (`yes`)  
- enter password `bandit0`  

Important:
- password input is hidden (no characters shown) — this is normal  

After successful login, you get access to a remote shell.

---

## ⚠️ Notes

- This is an introductory level — credentials are provided  
- After login, you are working on a remote machine  

---

## 🧩 Result

Successfully connected to the Bandit server via SSH.

---

## 📚 Commands Used

- `ssh`

---

## 🚫 Disclaimer

From the next levels, passwords must be found manually and should **not be published**. Only the solution approach is documented.
