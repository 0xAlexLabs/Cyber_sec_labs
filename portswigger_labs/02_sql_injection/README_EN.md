# 💉 SQL Injection — PortSwigger Web Security Academy

<p align="center">
  <img src="https://img.shields.io/badge/Platform-PortSwigger-red" />
  <img src="https://img.shields.io/badge/Topic-SQL%20Injection-blue" />
  <img src="https://img.shields.io/badge/Labs-15-success" />
  <img src="https://img.shields.io/badge/Progress-100%25-brightgreen" />
  <img src="https://img.shields.io/badge/Language-RU%20%7C%20EN-purple" />
</p>

<p align="right">
🇷🇺 <a href="./README.md"><b>Russian Version</b></a>
</p>

> 📂 Path: `cyber_sec_labs/portswigger_labs/sql_injection`  
> 📚 Section: SQL Injection  
> 🧪 Labs: 15  
> 📝 Reports: 30 — Russian and English versions for each lab

---

## 📚 Table of Contents

- [About](#-about)
- [Disclaimer](#️-disclaimer)
- [Progress](#-progress)
- [Learning Roadmap](#️-learning-roadmap)
- [Lab Map](#-lab-map)
- [Topics Covered](#-topics-covered)
- [SQL Injection Techniques](#-sql-injection-techniques)
- [Databases and Syntax](#-databases-and-syntax)
- [Tools](#-tools)
- [Walkthrough Format](#-walkthrough-format)
- [Directory Structure](#-directory-structure)
- [What You Will Learn](#-what-you-will-learn)
- [SQL Injection Prevention](#️-sql-injection-prevention)
- [Useful Links](#-useful-links)
- [Next Steps](#-next-steps)

---

## 📌 About

This directory contains walkthroughs for the **SQL Injection** labs from **PortSwigger Web Security Academy**.

The goal is not just to store final solutions, but to build a clear knowledge base:

- how to identify SQL Injection;
- why a specific payload works;
- how to choose the right exploitation technique based on application behavior;
- how to distinguish classic SQL Injection from Blind SQL Injection;
- how to adapt payloads for PostgreSQL, Oracle, MySQL, and Microsoft SQL Server;
- how to prevent SQL Injection in real applications.

These notes are written as practical learning material: each walkthrough focuses on the reasoning behind the attack, not only on the final payload.

---

## ⚠️ Disclaimer

These materials are for **educational purposes only**.

All actions were performed only inside the legal PortSwigger Web Security Academy lab environment. Do not use these techniques against real systems without explicit written authorization.

---

## 📊 Progress

```text
Section: SQL Injection
Labs: 15 / 15
Reports: 30
Languages: RU / EN
Status: completed
```

```text
███████████████ 15 / 15 — 100%
```

---

## 🗺️ Learning Roadmap

```text
SQL Injection
│
├── Basics
│   ├── Hidden Data
│   └── Login Bypass
│
├── UNION Attacks
│   ├── Column Count
│   ├── Text Column
│   ├── Data Retrieval
│   └── Single-Column Concatenation
│
├── Database Enumeration
│   ├── Database Version
│   └── information_schema
│
├── Blind SQL Injection
│   ├── Conditional Responses
│   ├── Conditional Errors
│   ├── Visible Error-Based SQLi
│   ├── Time-Based SQLi
│   └── OAST
│
├── XML Context
│   └── XML Entity Encoding
│
└── Prevention
    ├── Prepared Statements
    ├── Parameterized Queries
    └── Allow Lists
```

---

## 📋 Lab Map

| # | Lab | What it teaches | RU | EN |
|---|-----|-----------------|:--:|:--:|
| 001 | **SQL injection vulnerability in WHERE clause allowing retrieval of hidden data**<br><sub>Retrieving hidden data via SQL Injection</sub> | WHERE clause manipulation, `OR 1=1`, SQL comments, hidden data retrieval | [RU](./001_ru_sql_injection_hidden_data.md) | [EN](./001_en_sql_injection_hidden_data.md) |
| 002 | **SQL injection vulnerability allowing login bypass**<br><sub>Authentication bypass via SQL Injection</sub> | Password check bypass, SQL comments, authentication bypass logic | [RU](./002_ru_sql_login_bypass.md) | [EN](./002_en_sql_login_bypass.md) |
| 003 | **SQL injection UNION attack, determining the number of columns returned by the query**<br><sub>UNION attack: determining the number of columns</sub> | UNION SELECT, NULL placeholders, column count discovery | [RU](./003_ru_union_column_count.md) | [EN](./003_en_union_column_count.md) |
| 004 | **SQL injection UNION attack, finding a column containing text**<br><sub>UNION attack: finding a text-compatible column</sub> | Finding a string-compatible visible column for data output | [RU](./004_ru_union_text_column.md) | [EN](./004_en_union_text_column.md) |
| 005 | **SQL injection UNION attack, retrieving data from other tables**<br><sub>UNION attack: retrieving data from another table</sub> | Reading the users table and extracting username/password pairs | [RU](./005_ru_union_retrieve_users.md) | [EN](./005_en_union_retrieve_users.md) |
| 006 | **SQL injection UNION attack, retrieving multiple values in a single column**<br><sub>UNION attack: retrieving multiple values in one column</sub> | String concatenation, separators, single-column data extraction | [RU](./006_ru_union_single_column.md) | [EN](./006_en_union_single_column.md) |
| 007 | **SQL injection attack, querying the database type and version**<br><sub>Database type and version fingerprinting</sub> | Database fingerprinting, version retrieval, vendor-specific syntax | [RU](./007_ru_database_version_mysql_mssql.md) | [EN](./007_en_database_version_mysql_mssql.md) |
| 008 | **SQL injection attack, listing the database contents on non-Oracle databases**<br><sub>Database content enumeration on non-Oracle databases</sub> | information_schema, table discovery, column discovery, credential extraction | [RU](./008_ru_db_enumeration_non_oracle.md) | [EN](./008_en_db_enumeration_non_oracle.md) |
| 009 | **Blind SQL Injection with Conditional Responses**<br><sub>Blind SQLi through conditional responses</sub> | Boolean inference, response differences, Welcome back indicator, character-by-character extraction | [RU](./009_ru_blind_sqli_conditional_responses.md) | [EN](./009_en_blind_sqli_conditional_responses.md) |
| 010 | **Blind SQL Injection with Conditional Errors**<br><sub>Blind SQLi through conditional errors</sub> | Oracle CASE expressions, deliberate errors, error/no-error inference | [RU](./010_ru_blind_sqli_conditional_errors.md) | [EN](./010_en_blind_sqli_conditional_errors.md) |
| 011 | **Visible Error-Based SQL Injection**<br><sub>Visible error-based SQL Injection</sub> | CAST errors, verbose SQL error messages, direct data leakage through errors | [RU](./011_ru_visible_error_based_sqli.md) | [EN](./011_en_visible_error_based_sqli.md) |
| 012 | **Blind SQL Injection with Time Delays and Information Retrieval**<br><sub>Blind SQLi through time delays and data extraction</sub> | pg_sleep(), time-based inference, Burp Intruder, Cluster Bomb attack | [RU](./012_ru_blind_sqli_time_delays_and_information_retrieval.md) | [EN](./012_en_blind_sqli_time_delays_and_information_retrieval.md) |
| 013 | **Blind SQL Injection with Out-of-Band Interaction**<br><sub>Blind SQLi through OAST interaction</sub> | Burp Collaborator, DNS callbacks, out-of-band SQL execution confirmation | [RU](./013_ru_blind_sqli_oast.md) | [EN](./013_en_blind_sqli_oast.md) |
| 014 | **Blind SQL Injection with Out-of-Band Data Exfiltration**<br><sub>Blind SQLi through OAST data exfiltration</sub> | Password exfiltration through a Collaborator subdomain | [RU](./014_ru_blind_sqli_oast_data_exfiltration.md) | [EN](./014_en_blind_sqli_oast_data_exfiltration.md) |
| 015 | **SQL Injection with Filter Bypass via XML Encoding**<br><sub>SQL Injection in XML with WAF bypass</sub> | XML Entity Encoding, Hackvertor, weak WAF bypass | [RU](./015_ru_sql_injection_filter_bypass_xml_encoding.md) | [EN](./015_en_sql_injection_filter_bypass_xml_encoding.md) |

---

## 🧠 Topics Covered

### 1. Basic SQL Injection

The first labs explain the foundation of SQL Injection: user-controlled input reaches a SQL query, and the attacker can modify the query logic.

Key ideas:

- closing a string literal;
- adding conditions such as `OR 1=1`;
- using SQL comments to remove the rest of the original query;
- bypassing filters such as `released = 1`;
- bypassing login checks.

Core concept:

```text
If user-controlled input is inserted directly into SQL,
the user may be able to change the structure of the query.
```

---

### 2. UNION-Based SQL Injection

UNION attacks allow the tester to append a custom `SELECT` query to the original application query.

Covered topics:

- how `UNION SELECT` works;
- why the number of columns must match;
- why `NULL` is useful for column count testing;
- how to find a text-compatible column;
- how to retrieve data from the `users` table;
- how to concatenate multiple values into one output column.

Example:

```sql
UNION SELECT NULL,NULL,NULL--
```

If the number of columns matches the original query, the injected query executes successfully.

---

### 3. Database Enumeration

After confirming SQL Injection, the next step is often database enumeration.

Covered topics:

- identifying the database type;
- retrieving the database version;
- using `information_schema.tables`;
- using `information_schema.columns`;
- finding non-standard table names;
- finding non-standard column names;
- extracting credentials.

Example:

```sql
UNION SELECT NULL,table_name FROM information_schema.tables--
```

---

### 4. Blind SQL Injection

Blind SQL Injection is used when the application does not show SQL query results directly.

Instead, the tester uses indirect inference channels:

```text
TRUE / FALSE response difference
SQL error / no SQL error
Response delay / no delay
External DNS or HTTP interaction
```

Covered techniques:

- Conditional Responses;
- Conditional Errors;
- Visible Error-Based SQL Injection;
- Time-Based SQL Injection;
- Out-of-Band SQL Injection;
- Out-of-Band Data Exfiltration.

---

### 5. Time-Based SQL Injection

In time-based SQL Injection, the database communicates through delay.

Typical logic:

```text
TRUE  → delayed response
FALSE → immediate response
```

Example PostgreSQL payload:

```sql
SELECT CASE WHEN (condition) THEN pg_sleep(10) ELSE pg_sleep(0) END
```

This technique is useful when the page content does not visibly change and SQL errors are not displayed.

---

### 6. Out-of-Band SQL Injection

OAST means **Out-of-Band Application Security Testing**.

In these labs, the result does not come back through the original HTTP response. Instead, the vulnerable server or database performs an external DNS or HTTP interaction.

Typical chain:

```text
Browser
↓
Application
↓
Database
↓
DNS / HTTP request
↓
Burp Collaborator
```

This is especially useful when:

- the query is executed asynchronously;
- the HTTP response does not change;
- timing delays are not useful;
- direct SQL output is unavailable.

---

### 7. SQL Injection in XML and WAF Bypass

The final lab demonstrates that SQL Injection can exist not only in URLs, cookies, or form fields, but also inside XML request bodies.

Processing chain:

```text
HTTP Request
↓
WAF
↓
XML Parser
↓
SQL Engine
```

Core idea:

```text
The WAF sees an encoded XML payload,
while the SQL engine receives the decoded SQL syntax.
```

Hackvertor and XML Entity Encoding are used to bypass weak keyword-based filtering.

---

## 🧩 SQL Injection Techniques

- **Conditional Responses** — inference based on visible differences in application responses.
- **Conditional Errors** — inference through deliberately triggered SQL errors.
- **Visible Error-Based SQL Injection** — direct data extraction from verbose SQL error messages.
- **Time-Based SQL Injection** — extracting information by measuring response delays.
- **Out-of-Band SQL Injection (OAST)** — confirming SQL execution through external DNS or HTTP interactions.
- **Out-of-Band Data Exfiltration** — leaking sensitive data through an external channel, such as a Burp Collaborator subdomain.
- **XML Entity Encoding Bypass** — bypassing weak WAF rules using XML entities that are decoded after inspection.
- **Second-Order SQL Injection** — a stored payload is saved safely first, but later reused unsafely in another SQL query.
- **Prevention with Prepared Statements** — separating SQL code from user-controlled data.

---

## 🗄 Databases and Syntax

| Database | What is covered |
|----------|-----------------|
| PostgreSQL | `pg_sleep()`, `CAST()`, `LIMIT`, time-based SQLi, visible error-based SQLi |
| Oracle | `dual`, `SUBSTR()`, `TO_CHAR(1/0)`, `xmltype()`, `EXTRACTVALUE()` |
| MySQL | `@@version`, `#` comments, `-- ` comment behavior, `LIMIT` |
| Microsoft SQL Server | `@@version`, `WAITFOR DELAY`, theoretical OAST through `xp_dirtree` |
| Non-Oracle databases | `information_schema.tables`, `information_schema.columns` |

Payloads are often database-specific. A payload that works in PostgreSQL may fail in Oracle, MySQL, or Microsoft SQL Server.

---

## 🛠 Tools

- **Burp Proxy** — intercepting HTTP requests.
- **Burp Repeater** — manual payload testing.
- **Burp Intruder** — character brute-forcing and Blind SQLi automation.
- **Burp Collaborator** — OAST, DNS callbacks, HTTP callbacks.
- **Hackvertor** — payload transformation and encoding, including XML hex entities.
- **Burp Decoder** — URL encoding and decoding.
- **Browser Developer Tools** — request inspection and debugging.
- **PortSwigger SQL Injection Cheat Sheet** — payload reference for different database engines.

---

## 📝 Walkthrough Format

Each walkthrough explains not only **what to send**, but also **why it works**.

Most reports include:

- lab goal;
- short theory;
- step-by-step exploitation;
- payloads used;
- key payload breakdown;
- application behavior analysis;
- pentester mindset;
- common mistakes;
- mitigation recommendations;
- checklist;
- hidden credentials using `<details>`.

The goal is to make each report useful for later review, interview preparation, and technical English practice.

---

## 📂 Directory Structure

```text
sql_injection/
│
├── README.md
├── README_EN.md
│
├── 001_ru_sql_injection_hidden_data.md
├── 001_en_sql_injection_hidden_data.md
│
├── 002_ru_sql_login_bypass.md
├── 002_en_sql_login_bypass.md
│
├── ...
│
├── 015_ru_sql_injection_filter_bypass_xml_encoding.md
└── 015_en_sql_injection_filter_bypass_xml_encoding.md
```

---

## 🎓 What You Will Learn

After completing this section, you will be able to:

- identify SQL Injection manually;
- understand how backend SQL queries are constructed;
- fingerprint the database engine;
- build payloads for PostgreSQL, Oracle, MySQL, and Microsoft SQL Server;
- exploit UNION-based SQL Injection;
- exploit Blind SQL Injection using responses, errors, timing, and OAST;
- extract data from other tables;
- bypass weak WAF rules;
- understand second-order SQL Injection;
- explain why Prepared Statements prevent classic SQL Injection;
- describe practical mitigation strategies.

---

## 🛡️ SQL Injection Prevention

Primary defense:

```text
Prepared Statements / Parameterized Queries
```

Prepared Statements separate SQL code from user-controlled data. This prevents user input from changing the structure of the query.

Also important:

- never build SQL through string concatenation;
- use allow-lists for `ORDER BY`, table names, and column names;
- validate data types;
- restrict database user privileges;
- hide verbose SQL errors from users;
- monitor suspicious requests;
- restrict outbound DNS and HTTP requests from database servers;
- disable dangerous XML features where possible;
- do not rely only on a WAF.

Weak protection:

```text
Searching for dangerous words such as UNION or SELECT in raw requests.
```

This can often be bypassed using encoding, comments, case changes, or alternative syntax.

---

## 🔗 Useful Links

- PortSwigger SQL Injection: https://portswigger.net/web-security/sql-injection
- SQL Injection Cheat Sheet: https://portswigger.net/web-security/sql-injection/cheat-sheet
- Burp Suite: https://portswigger.net/burp
- Web Security Academy: https://portswigger.net/web-security

---

## 🚀 Next Steps

After SQL Injection, logical next sections are:

- Cross-Site Scripting (XSS);
- Authentication;
- Access Control;
- XXE;
- SSRF;
- Server-Side Template Injection;
- Web Cache Poisoning.

---

⬆ [Back to top](#-sql-injection--portswigger-web-security-academy)
