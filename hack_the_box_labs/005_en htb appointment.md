# 📘 Hack The Box: Appointment

> 🔗 https://app.hackthebox.com/machines/Appointment

---

## 🎯 Goal

Gain access to the web application and retrieve the flag by exploiting a vulnerability in the authentication mechanism.

---

## 🧠 Key Idea

This lab demonstrates a classic web vulnerability — **SQL Injection** — which allows the attacker to bypass a login form without knowing a valid password.

The core idea of the machine is:

- only a web service is exposed;
- the main attack surface is the login form;
- user input is handled unsafely on the backend;
- the injection changes the logic of the SQL query;
- this leads to **authentication bypass**.

In other words, there is no need for brute force or a complicated exploit here. The real task is to understand **how the application validates credentials** and then turn that logic against itself.

---

## 🔍 Walkthrough

### Step 1 — Scan the target

```bash
nmap -sC -sV <TARGET_IP>
```

### Why this works

`nmap` helps identify:
- open ports;
- exposed services;
- service versions.

The `-sC` flag runs the default NSE scripts.  
The `-sV` flag performs version detection.

### What we learn

The scan result shows:

- the host is up;
- only **80/tcp** is open;
- service: **Apache httpd 2.4.38 ((Debian))**;
- page title: **Login**.

### Why this matters

This is where proper pentester mindset begins:

- no SSH;
- no SMB;
- no FTP;
- therefore, **the only attack surface is the web application**.

After such an `nmap` result, there is no reason to split focus across other services. Everything points to HTTP.

---

### Step 2 — Initial web application analysis

Opening the target in a browser reveals a simple login form.

The page contains:

- a `username` field;
- a `password` field;
- a `remember me` checkbox.

### Why this matters

A login form is not just a page element — it is an access control mechanism.

For an attacker, this means:

- the application accepts user-controlled input;
- the backend compares that input against stored data;
- if the input is unsafely embedded into a SQL query, the logic can potentially be altered.

At this stage, the right question is not “should I brute-force directories?” but:

> How is the server validating the username and password?

---

### Step 3 — Building the SQL Injection hypothesis

A typical insecure backend query may look like this:

```sql
SELECT * FROM users
WHERE username = 'admin' AND password = '123456';
```

If the developer constructs the SQL statement through string concatenation, user input becomes part of the query itself.

### Why this is dangerous

If the attacker controls the content of the login fields, they may be able to:

- close the original string with a quote;
- inject their own logical condition;
- comment out the remainder of the query.

That is exactly how **SQL Injection** works.

### OWASP classification

According to the OWASP Top 10 (2021), this vulnerability belongs to:

> **A03:2021 – Injection**

---

### Step 4 — Exploiting the vulnerability

The following payload was entered into the `username` field:

```sql
' OR 1=1 -- 
```

Any value can be entered into the `password` field.

### Why this works

Assume the application builds a query like this:

```sql
SELECT * FROM users
WHERE username = '<username>' AND password = '<password>';
```

After the payload is inserted, the query logically becomes:

```sql
SELECT * FROM users
WHERE username = '' OR 1=1 -- ' AND password = 'anything';
```

Now break it down:

#### 1. `'`
Closes the original SQL string.

#### 2. `OR 1=1`
Adds a condition that is **always true**.

#### 3. `--`
Turns the rest of the line into a comment.

As a result, the following fragment:

```sql
' AND password = 'anything'
```

is ignored by the database engine.

### Result

The `WHERE` clause becomes true regardless of the password, and the application treats the login as successful.

This is **authentication bypass** — access is granted without valid credentials.

---

### Step 5 — Alternative admin login using a comment

One of the tasks also asks for a login as `admin` using a comment.

A payload such as the following can be used:

```sql
admin' -- 
```

### Why this works

This modifies the logic of the query into something like:

```sql
SELECT * FROM users
WHERE username = 'admin' -- ' AND password = '...';
```

Everything after `--` is ignored.  
The password check disappears, and only the username `admin` remains relevant.

---

### Step 6 — Retrieve the flag

After a successful login, the web application returns a page containing the flag.

## 🔐 Flag

<details>
<summary>Show flag</summary>

e3d0796d002a446c0e622226f42e9672

</details>

---

## 📋 Task Answers

<details>
<summary>Show answers</summary>

1. Structured Query Language  
2. SQL injection  
3. A03:2021-Injection  
4. Apache httpd 2.4.38 ((Debian))  
5. 443  
6. directory  
7. 404  
8. dir  
9. \#  
10. Congratulations  

</details>



The core issue is that user input is embedded into the SQL query without safe handling.

This allows an attacker to:

- influence the structure of the SQL statement;
- alter the authentication logic;
- bypass login checks;
- gain unauthorized access.

In short, user-controlled data is treated not only as data, but as executable SQL syntax.

---

## 🛡️ Recommendations

To prevent this class of issue, the application should:

- use **prepared statements / parameterized queries**;
- avoid building SQL queries through string concatenation;
- validate and sanitize input properly;
- enforce least privilege for the database account;
- avoid exposing verbose SQL errors to users;
- use defensive controls such as a WAF where appropriate.

---

## 🛠️ Checklist

- [ ] Scan the target with `nmap`
- [ ] Identify port 80 and the web service
- [ ] Open the login page
- [ ] Recognize the login form as the main attack surface
- [ ] Test the SQL Injection hypothesis
- [ ] Perform authentication bypass
- [ ] Retrieve the flag
- [ ] Document the vulnerability and exploitation logic

---

## 📚 What this machine teaches

Appointment is a very simple machine, but a very important one from a methodology perspective.

It teaches you to:

- take `nmap` results seriously;
- connect a port to a service and then to a likely attack path;
- see a login form as a potential attack surface;
- understand SQL Injection not as a magic string, but as a logic manipulation technique;
- document not only the steps, but also the reasoning behind them.

That is exactly the kind of foundation needed for future web pentesting work.
