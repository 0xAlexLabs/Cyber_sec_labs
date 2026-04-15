# 📘 Hack The Box: Redeemer

> 🔗 https://app.hackthebox.com/machines/Redeemer

---

## 🎯 Goal

Connect to the Redis server and retrieve the flag from the database.

---

## 🧠 Key Idea

This lab demonstrates a classic Redis misconfiguration:

- Redis is exposed to the network;
- no authentication is required;
- anyone can connect and read data.

Core concept: **enumeration + direct data access**.

---

## 🔍 Walkthrough

### Step 1 — Scan the target

```bash
nmap -Pn -sV <TARGET_IP>
```

### Why this works

`nmap` identifies:
- open ports;
- services;
- versions.

`-Pn` skips host discovery.  
`-sV` detects the service.

### What we learn

- `6379/tcp open`
- service `redis`

### Why this matters

Port **6379** is the default Redis port.  
If open → database is accessible.

---

### Step 2 — Understanding Redis

Redis is an **in-memory database**.

### Why this matters

- stores data in RAM;
- fast access;
- often misconfigured.

---

### Step 3 — Connect to Redis

```bash
redis-cli -h <TARGET_IP>
```

### Why this works

`redis-cli` is the official Redis client.

`-h` specifies remote host.

No authentication → immediate access.

---

### Step 4 — Gather information

```bash
info
```

### Why this works

Returns:
- server info;
- stats;
- database info.

### What we learn

- version: **5.0.7**
- database `db0`
- keys: **4**

---

### Step 5 — Enumerate keys

```bash
keys *
```

### Why this works

Lists all keys in the database.

Equivalent to `ls` for data.

---

### Step 6 — Extract data

```bash
get flag
```

### Why this works

Redis stores `key → value`.

`get` retrieves the value.

---

## 📋 Task Answers

<details>
<summary>Show answers</summary>

1. 6379  
2. redis  
3. In-memory Database  
4. redis-cli  
5. -h  
6. info  
7. 5.0.7  
8. select  
9. 4  
10. keys *

</details>

---

## 🔐 Flag

<details>
<summary>Show flag</summary>

03e1d2b376c37ab3f5319922053953eb

</details>

---

## 💥 Vulnerability

- Redis exposed to network  
- no authentication  
- direct data access  

This results in **data exposure without exploitation**.

---

## 🛡️ Recommendations

- restrict Redis access (localhost/firewall)
- enable authentication
- use TLS
- avoid storing sensitive data in plaintext

---

## 🛠️ Checklist

- [ ] Scan target  
- [ ] Identify port 6379  
- [ ] Detect Redis  
- [ ] Connect via redis-cli  
- [ ] Run info  
- [ ] Enumerate keys  
- [ ] Extract data  
- [ ] Retrieve flag
