# 📘 PortSwigger Lab: 2FA Simple Bypass

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-simple-bypass  
> 🎯 Topic: Authentication vulnerabilities — bypassing two-factor authentication

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🔍 Step-by-Step Solution](#steps)
- [🔬 Why the Attack Worked](#breakdown)
- [🧠 Pentester Mindset](#pentester)
- [❌ Common Mistakes](#mistakes)
- [🛡 Mitigation](#defense)
- [✅ Checklist](#checklist)

---

<a id="goal"></a>

## 🎯 Goal

Access the account page of user `carlos` using his valid username and password, but without having his two-factor authentication code.

Credentials:

```text
Your account:
wiener:peter

Victim account:
carlos:montoya
```

---

<a id="theory"></a>

## 🧠 Short Theory

**Two-factor authentication (2FA)** requires two different proofs of identity:

```text
1. A password — something the user knows
2. A code — something the user receives on a device
```

Correct flow:

```text
Correct password
        ↓
Enter 2FA code
        ↓
Correct code
        ↓
Access the account
```

In this lab, the server makes a mistake. After checking the password, it already treats the user as authenticated and does not verify completion of 2FA when the account page is opened.

---

<a id="idea"></a>

## 🧩 Core Idea

After valid credentials are submitted, the application redirects the user to a 2FA verification page.

Instead of entering the code, manually open:

```text
/my-account
```

If the server displays the account page, the second factor is enforced only by the visible login flow and not by the server when protected resources are requested.

Attack flow:

```text
carlos:montoya
        ↓
2FA verification page
        ↓
Manually open /my-account
        ↓
Access Carlos's account
```

---

<a id="steps"></a>

## 🔍 Step-by-Step Solution

### Step 1 — Log in to your own account

Use:

```text
Username: wiener
Password: peter
```

The lab sends the 2FA code to the built-in email client.

Enter the code, open the account page, and note its URL:

```text
/my-account
```

This is the protected page that will later be tested without completing the second authentication step.

---

### Step 2 — Log out

Click `Log out` to end the current session.

---

### Step 3 — Log in as Carlos

Use:

```text
Username: carlos
Password: montoya
```

The password is accepted and the application asks for a 2FA verification code.

However, we do not have access to Carlos's email.

---

### Step 4 — Skip the 2FA page

Do not enter a code.

Manually change the URL in the browser to:

```text
/my-account
```

---

### Step 5 — Verify access

The page loads and displays:

```text
My Account

Your username is: carlos
Your email is: carlos@carlos-montoya.net
```

This confirms that Carlos's account was accessed without completing 2FA.

The lab is solved.

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

After the correct password was submitted, the server had already created a valid session for Carlos.

The vulnerable logic may have looked like this:

```python
if username_and_password_are_correct:
    create_authenticated_session()
    redirect_to_2fa_page()
```

The `/my-account` page only checked whether a session existed:

```python
if user_has_session:
    show_account_page()
```

The server did not check a separate state such as:

```text
2FA completed = true
```

As a result, the second authentication step could be skipped completely.

A secure implementation should create a fully authenticated session only after the 2FA code has been successfully verified.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

When an application uses a multi-step login process, do not immediately try to brute-force the verification code.

First ask:

```text
Can this step be skipped?
```

Useful testing sequence:

```text
1. Identify the URL of a protected page.
2. Submit valid username and password.
3. Stop at the 2FA page.
4. Manually request the protected page.
5. Check whether the server verifies completion of 2FA.
```

Main lesson:

```text
The presence of a 2FA form does not prove
that the server actually enforces 2FA.
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### 1. Trying to brute-force the code immediately

The verification code is not needed in this lab because the entire second step can be skipped.

### 2. Not identifying the account page URL

It is useful to log in to your own account first and note the URL of the protected page.

### 3. Trusting the visible login flow

A browser page asking for a code does not prove that the server blocks direct access to other resources.

### 4. Confusing user interface flow with security enforcement

Security must be enforced on the server, not only by the order of pages shown in the browser.

---

<a id="defense"></a>

## 🛡 Mitigation

Proper defenses include:

- creating only a temporary restricted session after password verification;
- storing a separate server-side state showing whether 2FA was completed;
- checking this state on every protected request;
- creating a fully authenticated session only after a correct 2FA code;
- blocking access to `/my-account`, `/admin`, and protected APIs before 2FA completion;
- redirecting incomplete sessions back to the verification page.

Correct flow:

```text
Correct password
        ↓
Temporary session
        ↓
Correct 2FA code
        ↓
Fully authenticated session
        ↓
Access to /my-account
```

---

<a id="checklist"></a>

## ✅ Checklist

- [x] Logged in as `wiener`.
- [x] Received and entered the personal 2FA code.
- [x] Identified the protected URL `/my-account`.
- [x] Logged out of the personal account.
- [x] Submitted `carlos:montoya`.
- [x] Reached the 2FA verification page.
- [x] Did not enter a 2FA code.
- [x] Manually opened `/my-account`.
- [x] Accessed Carlos's account.
- [x] Solved the lab.

---

# ⬆ Back to Top

[Return to contents](#top)
