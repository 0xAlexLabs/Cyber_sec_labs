# 📘 PortSwigger Lab: CSRF where token is not tied to user session

<a id="top"></a>

🔗 **Official Lab:** https://portswigger.net/web-security/learning-paths/csrf/csrf-common-flaws-in-csrf-token-validation/csrf/bypassing-token-validation/lab-csrf-token-not-tied-to-user-session  
🎯 **Topic:** CSRF — tokens not integrated into the site's session handling system  
🧪 **Difficulty:** Practitioner  
✅ **Status:** Solved  
🛠 **Tools:** Burp Suite (Proxy, Repeater), Exploit Server, two browser sessions  
🔑 **Credentials:** `wiener:peter` / `carlos:montoya`

---

## 📑 Contents
- [📘 PortSwigger Lab: CSRF where token is not tied to user session](#-portswigger-lab-csrf-where-token-is-not-tied-to-user-session)
  - [📑 Contents](#-contents)
  - [🎯 Goal](#-goal)
  - [🧠 Short Theory](#-short-theory)
  - [🧩 Core Idea](#-core-idea)
  - [🗺 Attack Scheme](#-attack-scheme)
  - [🔍 Step 1 — Analyze Legitimate Request](#-step-1--analyze-legitimate-request)
  - [🔍 Step 2 — Test Token Across Sessions](#-step-2--test-token-across-sessions)
  - [🔍 Step 3 — Confirm Token Is Not Session-Bound](#-step-3--confirm-token-is-not-session-bound)
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
The email change functionality is protected by a CSRF token, but the tokens are **not integrated into the site's session handling system**. The task:

1. Use the exploit server to host an HTML page that performs a CSRF attack.
2. Change the victim's email address via CSRF.

Two accounts are provided to help design the attack: `wiener:peter` and `carlos:montoya`.

<a id="theory"></a>
## 🧠 Short Theory
CSRF (Cross-Site Request Forgery) is an attack where the attacker forces the victim's browser to perform an unwanted action on a vulnerable site where the victim is authenticated.

A CSRF token is only effective when it is **bound to the victim's session**. A common implementation flaw is a global token pool: the server issues tokens tied to nothing (or stored globally), so any valid token issued to **any** user is accepted for **any** session. The attacker simply logs in with their own account, takes their own valid CSRF token, and hardcodes it into the exploit — the server cannot tell that the token was issued to someone else.

<a id="idea"></a>
## 🧩 Core Idea
Obtain a valid CSRF token from your **own** account and embed it in the exploit.

- The attacker can log in to the target site as a normal user and receive a genuine CSRF token.
- Since tokens are not tied to sessions, the victim's server accepts the attacker's token.
- The token is hardcoded into the exploit HTML — no need to leak the victim's token.
- Auto-submit via JavaScript makes the attack invisible to the victim.

<a id="attack-scheme"></a>
## 🗺 Attack Scheme
1. **Recon:** Intercept `POST /my-account/change-email` from account #1 (wiener), capture the csrf token.
2. **Verify:** Log in as account #2 (carlos) and swap in wiener's token → accepted → token is not session-bound.
3. **Exploit:** Build an HTML form POST to `/my-account/change-email` with a hardcoded valid csrf token.
4. **Test:** View exploit on yourself → email changed.
5. **Deliver:** Deliver to victim → lab solved.

⚠️ Note: tokens are **single-use**, so the token embedded in the exploit must be a fresh, unused one at delivery time.

<a id="step1"></a>
## 🔍 Step 1 — Analyze Legitimate Request
Logged in as `wiener:peter` in Burp's browser, submitted the "Update email" form, and intercepted the resulting request in Burp Proxy. Noted the `csrf` value, then **dropped** the request (do not consume the token).

```http
POST /my-account/change-email HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_WIENER
Content-Type: application/x-www-form-urlencoded

email=test@test.com&csrf=CSRF_TOKEN_WIENER
```

<a id="step2"></a>
## 🔍 Step 2 — Test Token Across Sessions
Opened a private/incognito browser window, logged in as the second account `carlos:montoya`, and sent carlos's "Update email" request into Burp Repeater.

In Repeater, **replaced carlos's csrf token with wiener's token** (`CSRF_TOKEN_WIENER`) and sent the request.

<a id="step3"></a>
## 🔍 Step 3 — Confirm Token Is Not Session-Bound
**Result:** the request was **accepted** — carlos's session successfully used wiener's CSRF token.

*Conclusion: tokens are stored in a global pool, not tied to the user's session. Any valid token works for any user — the attacker can simply embed their own token in the exploit.*

<a id="step4"></a>
## 🔍 Step 4 — Build CSRF Exploit
Built the exploit using a fresh (unused) CSRF token obtained from the attacker's own account, since tokens are single-use:

```html
<form method="POST" action="https://LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="csrf" value="CSRF_TOKEN_WIENER">
    <input type="hidden" name="email" value="attacker@web-security-academy.net">
</form>
<script>
    document.forms[0].submit();
</script>
```

*Important: the `csrf` input contains the attacker's own valid token — the server does not check which session it was issued to.*

<a id="step5"></a>
## 🔍 Step 5 — Test Exploit on Yourself
Pasted the HTML into the exploit server "Body" field, stored it, and clicked "View exploit". Confirmed the POST succeeds with the hardcoded token and the email changes. Then changed the email in the exploit so it doesn't match the current one.

⚠️ If the token was already consumed during testing, request a **fresh one** (log in again / load the account page) before delivering.

<a id="step6"></a>
## 🔍 Step 6 — Deliver Exploit to Victim
Stored the exploit with a fresh token and clicked "Deliver to victim". The victim's email was changed → lab solved.

<a id="requests"></a>
## 📨 Example Requests
**Legitimate request from wiener (with own token)**
```http
POST /my-account/change-email HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_WIENER
Content-Type: application/x-www-form-urlencoded

email=test@test.com&csrf=CSRF_TOKEN_WIENER
```

**Cross-session test: carlos's session + wiener's token**
```http
POST /my-account/change-email HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_CARLOS
Content-Type: application/x-www-form-urlencoded

email=test@test.com&csrf=CSRF_TOKEN_WIENER
```

<a id="response"></a>
## 📥 Example Result
**Cross-Session Token Test**
- carlos session + wiener's valid token → 302 Redirect (success)  
→ Token accepted despite belonging to a different user's session

**Exploit**
- Victim's session + attacker's hardcoded token → 302 Redirect (success)  
→ Email changed with a token that was never issued to the victim

<a id="attack-chain"></a>
## 🧾 Complete Attack Chain
1. Login as wiener → capture csrf token from `POST /my-account/change-email` (drop the request).
2. Incognito → login as carlos → Repeater → swap in wiener's token → accepted (token not session-bound).
3. Exploit Server → HTML form with hardcoded attacker token + auto-submit.
4. View exploit → test on yourself → email changed.
5. Refresh token (single-use), change email in exploit.
6. Deliver to victim → victim's email changed → **Solved**.

<a id="breakdown"></a>
## 🔬 Why the Attack Worked
- **Tokens not session-bound:** the server stores CSRF tokens in a global pool; any valid token is accepted for any session.
- **Attacker is also a user:** the site issues genuine tokens to the attacker's own account — no token theft required.
- **No binding to request context:** nothing cryptographically links the token to the victim's session or the specific action.
- **Single-use, but predictable:** the attacker can always obtain a fresh valid token right before delivering the exploit.
- **Auto-submit:** JavaScript submits the form instantly, the victim does not notice the attack.

<a id="pentester"></a>
## 🧠 Pentester Mindset
1. **Test tokens between two own accounts:** if account A's token works for account B, tokens are not session-bound.
2. **Ask: where are tokens stored?** Session-bound vs global pool changes the whole attack class.
3. **No need to leak the victim's token:** if tokens are interchangeable, your own token is enough.
4. **Watch token lifetime:** single-use tokens require a fresh value at delivery time — don't burn it in testing.
5. **Keep the exploit minimal:** form + hidden inputs + auto-submit is all you need.

<a id="mistakes"></a>
## ❌ Common Mistakes
- **Burning the token during testing:** tokens are single-use — if you test "View exploit" with the same token, it may be consumed; request a fresh one before "Deliver to victim".
- **Trying to steal the victim's token:** unnecessary here — the flaw is that any valid token works.
- **Dropping vs forwarding Step 1 request:** if you forward it, the token may be consumed; the lab expects you to note the value and drop it.
- **Forgot to change the email in the exploit:** if the address matches the current one, no change occurs.
- **Forgot auto-submit:** without `<script>` the form won't submit automatically.

<a id="defense"></a>
## 🛡 Mitigation
- **Bind the CSRF token to the user's session** — store the token inside the session object and validate the pair (session, token).
- Prefer **per-session tokens** derived from session-specific data (e.g. HMAC over the session ID) rather than a global pool.
- Use framework-level CSRF protection that ties tokens to sessions by default.
- **SameSite cookies** (Lax/Strict) as defense in depth.
- Check **Origin/Referer** headers for state-changing requests.

<a id="checklist"></a>
## ✅ Checklist
- [ ] Captured csrf token from wiener's request (dropped the request)
- [ ] Logged in as carlos in a separate session
- [ ] Confirmed: carlos's session accepts wiener's token
- [ ] Built HTML exploit with attacker's hardcoded token + auto-submit
- [ ] View exploit → tested on yourself → email changed
- [ ] Email in exploit differs from current
- [ ] Fresh token used before delivery (single-use)
- [ ] Deliver to victim → lab solved
- [ ] Status: **Solved**

<a id="conclusion"></a>
## 🧾 Conclusion
The lab demonstrates that a CSRF token is only a defense when it is cryptographically or structurally bound to the victim's session. Here the tokens live in a global pool: any valid token is accepted for any user, so the attacker simply embeds their own token into the exploit. The key lesson: token presence and validity are not enough — the server must verify that the token belongs to the requester's session.

⬆ [Back to top](#top)