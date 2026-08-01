# 📘 PortSwigger Lab: Username Enumeration via Subtly Different Responses

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-subtly-different-responses  
> 📄 Candidate usernames: https://portswigger.net/web-security/authentication/auth-lab-usernames  
> 📄 Candidate passwords: https://portswigger.net/web-security/authentication/auth-lab-passwords  
> 🎯 Topic: Authentication vulnerabilities — Username Enumeration via subtle response differences + Password Brute Force

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🔍 Step 1 — Find the Login Request](#step1)
- [🔍 Step 2 — Configure Intruder](#step2)
- [🔍 Step 3 — Configure Grep - Extract](#step3)
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

Identify a valid username using a subtle difference in the login error message, brute-force this user's password, and access the account page.

---

<a id="theory"></a>

## 🧠 Short Theory

This lab is similar to the previous username enumeration lab, but the difference in the responses is not obvious.

Previously, the application clearly returned different messages:

```text
Invalid username
Incorrect password
```

In this lab, the message looks almost identical:

```text
Invalid username or password.
```

However, for a valid username, the response contains a tiny difference:

```text
Invalid username or password 
```

Difference:

```text
Normal response:     period at the end
Different response: trailing space instead of a period
```

This is very hard to notice manually, so Burp Intruder's **Grep - Extract** feature is used.

---

<a id="idea"></a>

## 🧩 Core Idea

The server attempts to hide username enumeration by returning a generic error message:

```text
Invalid username or password.
```

However, different code paths generate the message slightly differently.

For a non-existent username:

```html
<p class=is-warning>Invalid username or password.</p>
```

For an existing username with an incorrect password:

```html
<p class=is-warning>Invalid username or password </p>
```

This subtle difference allows us to identify a valid username.

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

Send the request to Intruder:

```text
Right click → Send to Intruder
```

---

<a id="step2"></a>

## 🔍 Step 2 — Configure Intruder

In the first stage, only the `username` parameter is tested.

Attack type:

```text
Sniper
```

Payload position:

```http
username=§test§&password=test
```

Keep the password static.

In Payloads, paste the username list:

```text
https://portswigger.net/web-security/authentication/auth-lab-usernames
```

Do not start the attack yet. First, configure response extraction.

---

<a id="step3"></a>

## 🔍 Step 3 — Configure Grep - Extract

Go to:

```text
Intruder → Settings → Grep - Extract → Add
```

In the response preview, find the error message:

```text
Invalid username or password.
```

Highlight only the error text, without HTML tags:

```text
Invalid username or password.
```

Burp automatically configures the extraction boundaries. Start the attack after saving the rule.

Why this matters:

```text
Grep - Extract places the error message in a separate column.
This makes a one-character difference much easier to spot.
```

---

<a id="step4"></a>

## 🔍 Step 4 — Identify a Valid Username

When the attack finishes, sort the results by the extracted error message column.

Most responses contain:

```text
Invalid username or password.
```

One response is different:

```text
Invalid username or password 
```

In HTML, it appeared as:

```html
<p class=is-warning>Invalid username or password </p>
```

There is no period at the end. Instead, there is a trailing space.

This means:

```text
The username exists, but the password is incorrect.
```

Identified username:

<details>
<summary>🔑 Show identified username</summary>

```text
pi
```

</details>

---

<a id="step5"></a>

## 🔍 Step 5 — Brute-force the Password

Now keep the identified username fixed and brute-force only the password:

```http
username=pi&password=§test§
```

In Payloads, paste the password list:

```text
https://portswigger.net/web-security/authentication/auth-lab-passwords
```

Start the attack.

A successful password is identified by:

```http
302 Found
```

This usually indicates a successful login and redirect to the account page.

---

<a id="step6"></a>

## 🔍 Step 6 — Log in to the Account

Use the discovered credentials on the login page.

<details>
<summary>🔑 Show discovered credentials</summary>

```text
Username: pi
Password: 1111
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
username=pi&password=§candidate_password§
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

The developer attempted to prevent username enumeration by returning the same generic message:

```text
Invalid username or password.
```

However, the actual messages differed by one character.

For a non-existent username:

```text
Invalid username or password.
```

For an existing username:

```text
Invalid username or password 
```

The application still leaked its internal logic:

```text
one code path → user not found
another code path → user found, password incorrect
```

Even if the difference is almost invisible, Burp can detect it using `Grep - Extract`.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

Correct thought process:

```text
1. Check whether responses are truly identical.
2. Do not trust visual inspection alone.
3. Extract the error message using Grep - Extract.
4. Sort the extracted values.
5. Find the response that differs by one character.
6. Fix the username.
7. Brute-force the password for that username.
```

Main lesson:

```text
If responses look the same, it does not mean they are actually identical.
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### 1. Relying only on visual inspection

A single trailing space is easy to miss in the browser or response viewer.

### 2. Looking only at Length

Response length can change because of unrelated page content. In this lab, `Grep - Extract` is more reliable.

### 3. Highlighting HTML together with the message

In Grep - Extract, highlight only the error text:

```text
Invalid username or password.
```

not the entire `<p>` tag.

### 4. Starting with Cluster Bomb

Full brute force can work, but it is less efficient and misses the point of the lab.

---

<a id="defense"></a>

## 🛡 Mitigation

Proper defenses:

- use exactly the same error message for all login failures;
- generate the error message in one shared code path;
- avoid differences in punctuation, whitespace, response length, status codes, and headers;
- implement rate limiting;
- use account lockout or progressive delays;
- use MFA;
- log large-scale login attempts;
- test authentication responses for exact equality.

Weak defense:

```text
Two almost identical messages generated by different code paths.
```

Even one trailing space can become a side channel.

---

<a id="checklist"></a>

## ✅ Checklist

- [x] `POST /login` request found.
- [x] Request sent to Intruder.
- [x] Username tested separately.
- [x] `Grep - Extract` configured.
- [x] Error message extracted.
- [x] Response with trailing space instead of period found.
- [x] Valid username identified.
- [x] Password tested separately.
- [x] `302 Found` response identified.
- [x] Logged in to the account.
- [x] Lab solved.

---

# ⬆ Back to Top

[Return to contents](#top)
