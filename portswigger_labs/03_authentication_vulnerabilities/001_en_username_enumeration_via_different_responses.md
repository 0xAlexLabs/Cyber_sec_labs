# 📘 PortSwigger Lab: Username Enumeration via Different Responses

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/authentication/lab-username-enumeration-via-different-responses  
> 📄 Candidate usernames: https://portswigger.net/web-security/authentication/auth-lab-usernames  
> 📄 Candidate passwords: https://portswigger.net/web-security/authentication/auth-lab-passwords  
> 🎯 Topic: Authentication vulnerabilities — Username Enumeration + Password Brute Force

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🔍 Step 1 — Find the Login Request](#step1)
- [🔍 Step 2 — Enumerate Usernames](#step2)
- [🔍 Step 3 — Identify a Valid Username](#step3)
- [🔍 Step 4 — Brute-force the Password](#step4)
- [🔍 Step 5 — Log in to the Account](#step5)
- [🧩 Data Used](#payloads)
- [🔬 Why the Attack Worked](#breakdown)
- [🧠 Pentester Mindset](#pentester)
- [❌ Common Mistakes](#mistakes)
- [🛡 Mitigation](#defense)
- [✅ Checklist](#checklist)

---

<a id="goal"></a>

## 🎯 Goal

Enumerate a valid username based on different login responses, brute-force this user's password, and access the account page.

---

<a id="theory"></a>

## 🧠 Short Theory

**Username Enumeration** occurs when an application gives different responses for a non-existent username and an existing username with an incorrect password.

For example:

```text
Invalid username
```

and:

```text
Incorrect password
```

For a normal user, these are just error messages. For an attacker, they reveal useful information:

```text
If the response changes, the username probably exists.
```

After identifying a valid username, the attacker no longer needs to brute-force every `username:password` pair. They can brute-force only the password for the confirmed user.

---

<a id="idea"></a>

## 🧩 Core Idea

The lab has two stages:

```text
1. Enumerate usernames using a fixed password.
2. Brute-force the password for the identified username.
```

This is much more efficient than brute-forcing all combinations at once.

```text
100 usernames × 100 passwords = 10,000 requests
100 usernames + 100 passwords = 200 requests
```

---

<a id="step1"></a>

## 🔍 Step 1 — Find the Login Request

Open the login page and submit any invalid credentials:

```text
username=test
password=test
```

In Burp Suite, go to:

```text
Proxy → HTTP history
```

Find the request:

```http
POST /login
```

Example request body:

```http
username=test&password=test
```

Send it to Intruder:

```text
Right click → Send to Intruder
```

---

<a id="step2"></a>

## 🔍 Step 2 — Enumerate Usernames

In Intruder, it is better to start with **Sniper**, not Cluster Bomb.

Set a payload position only on the `username` parameter:

```http
username=§test§&password=test
```

Keep the password static, for example:

```text
test
```

In Payloads, paste the PortSwigger username list:

```text
Candidate usernames
https://portswigger.net/web-security/authentication/auth-lab-usernames
```

Start the attack.

---

<a id="step3"></a>

## 🔍 Step 3 — Identify a Valid Username

When the attack finishes, inspect the Intruder results.

Most responses contain the same error:

```text
Invalid username
```

One response is different. In this lab, the difference can be detected using:

```text
Length
```

or the response body:

```text
Incorrect password
```

This means:

```text
The username exists, but the password is incorrect.
```

Identified username:

<details>
<summary>🔑 Show identified username</summary>

```text
argentina
```

</details>

---

<a id="step4"></a>

## 🔍 Step 4 — Brute-force the Password

Now modify the request.

Keep the identified username fixed:

```http
username=argentina&password=§test§
```

Place the payload position only on the `password` parameter.

In Payloads, paste the password list:

```text
Candidate passwords
https://portswigger.net/web-security/authentication/auth-lab-passwords
```

Start the attack.

This time, the key indicator is not the response length, but a successful login. It usually appears as:

```http
302 Found
```

or a redirect to:

```text
/my-account
```

Incorrect passwords usually return:

```http
200 OK
```

---

<a id="step5"></a>

## 🔍 Step 5 — Log in to the Account

Use the discovered credentials in the login form.

<details>
<summary>🔑 Show discovered credentials</summary>

```text
Username: argentina
Password: 111111
```

</details>

After accessing the account page, the lab is solved.

---

<a id="payloads"></a>

## 🧩 Data Used

Username list:

```text
https://portswigger.net/web-security/authentication/auth-lab-usernames
```

Password list:

```text
https://portswigger.net/web-security/authentication/auth-lab-passwords
```

First attack:

```http
username=§candidate_username§&password=test
```

Second attack:

```http
username=argentina&password=§candidate_password§
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

The application returned different messages for different login failure cases.

For a non-existent username:

```text
Invalid username
```

For an existing username with an incorrect password:

```text
Incorrect password
```

This created a side channel. The application did not directly disclose the list of users, but response differences made it possible to infer a valid username.

After identifying a valid username, the attack became much smaller: instead of brute-forcing all possible combinations, only one user's password had to be tested.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

Correct thought process:

```text
1. Do not start with full brute force immediately.
2. Check whether invalid username and invalid password responses differ.
3. If they differ, perform username enumeration first.
4. Then brute-force only the password for the valid username.
5. Identify successful login by a 302 redirect or navigation to /my-account.
```

The key skill in this lab is noticing small differences:

```text
different error text
different response length
different status code
different redirect behavior
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### 1. Starting with Cluster Bomb

This can work, but it is inefficient:

```text
100 × 100 = 10,000 requests
```

It is better to enumerate the username first:

```text
100 + 100 = 200 requests
```

### 2. Looking only at status code during username enumeration

During username enumeration, the status code is often the same:

```http
200 OK
```

You need to inspect `Length` and the response body.

### 3. Looking only at Length during password brute-force

During password brute-force, the important indicator is usually:

```http
302 Found
```

This indicates a successful login.

---

<a id="defense"></a>

## 🛡 Mitigation

Proper defenses:

- return the same message for invalid username and invalid password;
- use a generic message such as `Invalid username or password`;
- implement rate limiting;
- add account lockout or progressive delays after failed attempts;
- use MFA;
- log suspicious login attempts;
- detect large-scale username/password guessing;
- avoid leaking account existence through response text, response length, or different status codes.

Weak behavior:

```text
Invalid username
Incorrect password
```

These messages help attackers enumerate valid users.

---

<a id="checklist"></a>

## ✅ Checklist

- [x] `POST /login` request found.
- [x] Request sent to Intruder.
- [x] Username tested separately.
- [x] Different response identified.
- [x] Valid username found.
- [x] Password tested separately.
- [x] `302 Found` response identified.
- [x] Logged in to the account.
- [x] Lab solved.

---

# ⬆ Back to Top

[Return to contents](#top)
