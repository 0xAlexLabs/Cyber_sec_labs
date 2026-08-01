# 🔐 Authentication — PortSwigger Web Security Academy

<p align="center">
  <img src="https://img.shields.io/badge/Platform-PortSwigger-red" />
  <img src="https://img.shields.io/badge/Topic-Authentication-blue" />
  <img src="https://img.shields.io/badge/Labs-11-success" />
  <img src="https://img.shields.io/badge/Progress-100%25-brightgreen" />
  <img src="https://img.shields.io/badge/Language-RU%20%7C%20EN-purple" />
</p>

<p align="right">
🇷🇺 <a href="./README.md"><b>Russian Version</b></a>
</p>

> 📂 Path: `cyber_sec_labs/portswigger_labs/authentication`  
> 📚 Section: Authentication vulnerabilities  
> 🧪 Labs: 11  
> 📝 Reports: 22 — Russian and English versions for each lab

---

## 📚 Table of Contents

- [About](#-about)
- [Disclaimer](#️-disclaimer)
- [Progress](#-progress)
- [Learning Roadmap](#️-learning-roadmap)
- [Lab Map](#-lab-map)
- [Topics Covered](#-topics-covered)
- [Authentication Attack Techniques](#-authentication-attack-techniques)
- [Attack Surfaces](#-attack-surfaces)
- [Tools](#-tools)
- [Walkthrough Format](#-walkthrough-format)
- [Directory Structure](#-directory-structure)
- [What You Will Learn](#-what-you-will-learn)
- [Authentication Defense](#️-authentication-defense)
- [Useful Links](#-useful-links)
- [Next Steps](#-next-steps)

---

## 📌 About

This directory contains detailed walkthroughs for the **Authentication vulnerabilities** labs from **PortSwigger Web Security Academy**.

The purpose is not merely to preserve final solutions. It is to build a practical authentication-testing methodology:

- identify username enumeration through text, length, status, timing, and lockout behavior;
- analyze brute-force protection, rate limiting, and account-lock logic;
- verify whether the server truly enforces the second factor;
- examine the binding between factor one, the MFA code, the session, and the target user;
- reverse-engineer persistent authentication cookies;
- distinguish Base64 encoding from cryptographic protection;
- analyze password-reset flows, reset tokens, and proxy-header trust;
- explain each vulnerability in terms of backend logic;
- propose practical defensive controls.

Each walkthrough is designed for revision, interview preparation, technical-English practice, and the development of a repeatable web-pentest knowledge base.

---

## ⚠️ Disclaimer

These materials are for **educational purposes only**.

All actions were performed exclusively inside the legal PortSwigger Web Security Academy lab environment. Do not use brute force, token interception, cookie manipulation, or password-reset attacks against real systems without explicit written authorization.

---

## 📊 Progress

```text
Section: Authentication vulnerabilities
Labs: 11 / 11
Reports: 22
Languages: RU / EN
Status: completed
```

```text
███████████ 11 / 11 — 100%
```

---

## 🗺️ Learning Roadmap

```text
Authentication vulnerabilities
│
├── Password-Based Authentication
│   ├── Username Enumeration
│   │   ├── Different Responses
│   │   ├── Subtle Differences
│   │   ├── Response Timing
│   │   └── Account Lock
│   │
│   └── Brute-Force Protection
│       ├── IP Block
│       ├── Counter Reset Logic
│       ├── Rate Limiting
│       └── Intruder Attack Types
│
├── Multi-Factor Authentication
│   ├── Simple 2FA Bypass
│   ├── Server-Side MFA State
│   ├── Identity Binding
│   └── MFA Code Brute Force
│
├── Persistent Authentication
│   ├── Stay-Logged-In Cookie
│   ├── Base64
│   ├── MD5
│   ├── Cookie Forgery
│   └── Offline Cracking
│
├── Password Reset
│   ├── Broken Reset Logic
│   ├── Token-to-User Binding
│   ├── Hidden Parameters
│   ├── Host Header Injection
│   └── Reset Link Poisoning
│
└── Prevention
    ├── Uniform Responses
    ├── Strong Rate Limiting
    ├── Secure MFA State
    ├── Random Persistent Tokens
    └── Trusted Reset URLs
```

---

## 📋 Lab Map

| # | Lab | What it teaches | RU | EN |
|---|-----|-----------------|:--:|:--:|
| 001 | **Username enumeration via different responses**<br><sub>Перечисление пользователей по различающимся ответам</sub> | different login errors, two-stage brute force, Status and Length analysis | [RU](./001_ru_username_enumeration_via_different_responses.md) | [EN](./001_en_username_enumeration_via_different_responses.md) |
| 002 | **Username enumeration via subtly different responses**<br><sub>Перечисление пользователей по едва заметным различиям</sub> | Grep - Extract, missing-period and trailing-space detection, near-identical response comparison | [RU](./002_ru_username_enumeration_via_subtly_different_responses.md) | [EN](./002_en_username_enumeration_via_subtly_different_responses.md) |
| 003 | **Username enumeration via response timing**<br><sub>Перечисление пользователей по времени ответа</sub> | timing side channel, long-password amplification, `X-Forwarded-For` lockout bypass | [RU](./003_ru_username_enumeration_via_response_timing.md) | [EN](./003_en_username_enumeration_via_response_timing.md) |
| 004 | **Broken brute-force protection, IP block**<br><sub>Обход защиты от brute force через IP block</sub> | counter reset with valid logins, Pitchfork, Resource Pool = 1 | [RU](./004_ru_broken_bruteforce_protection_ip_block.md) | [EN](./004_en_broken_bruteforce_protection_ip_block.md) |
| 005 | **Username enumeration via account lock**<br><sub>Перечисление пользователей через блокировку аккаунта</sub> | account-lock side channel, Cluster Bomb, Null Payloads | [RU](./005_ru_username_enumeration_via_account_lock.md) | [EN](./005_en_username_enumeration_via_account_lock.md) |
| 006 | **2FA simple bypass**<br><sub>Простой обход двухфакторной аутентификации</sub> | skipping `/login2`, direct access to `/my-account`, missing server-side enforcement | [RU](./006_ru_2fa_simple_bypass.md) | [EN](./006_en_2fa_simple_bypass.md) |
| 007 | **2FA broken logic**<br><sub>Нарушенная логика двухфакторной аутентификации</sub> | tampering with `verify`, victim-code generation, MFA-code brute force | [RU](./007_ru_2fa_broken_logic.md) | [EN](./007_en_2fa_broken_logic.md) |
| 008 | **Brute-forcing a stay-logged-in cookie**<br><sub>Подбор cookie постоянного входа</sub> | Base64, MD5, Payload Processing, persistent-cookie forgery | [RU](./008_ru_brute_forcing_stay_logged_in_cookie.md) | [EN](./008_en_brute_forcing_stay_logged_in_cookie.md) |
| 009 | **Offline password cracking**<br><sub>Офлайн-восстановление пароля</sub> | Stored XSS, cookie theft, MD5 extraction, offline cracking | [RU](./009_ru_offline_password_cracking.md) | [EN](./009_en_offline_password_cracking.md) |
| 010 | **Password reset broken logic**<br><sub>Нарушенная логика сброса пароля</sub> | hidden `username` tampering, empty token, resetting the victim's password | [RU](./010_ru_password_reset_broken_logic.md) | [EN](./010_en_password_reset_broken_logic.md) |
| 011 | **Password reset poisoning via middleware**<br><sub>Отравление ссылки сброса пароля через middleware</sub> | `X-Forwarded-Host`, absolute reset URLs, token interception | [RU](./011_ru_password_reset_poisoning_via_middleware.md) | [EN](./011_en_password_reset_poisoning_via_middleware.md) |

---

## 🧠 Topics Covered

### 1. Password-Based Authentication

A basic login form looks simple:

```text
username + password
```

For a pentester, however, the error message is only one observable signal. The complete result matters:

```text
HTTP status
Response body
Response length
Response time
Redirect target
Set-Cookie
Account lock behavior
```

Core principle:

```text
Do not compare only the message.
Compare the full HTTP response and application state.
```

---

### 2. Username Enumeration

Username enumeration confirms that an account exists before password guessing begins.

This section covers four channels:

```text
1. Explicitly different error messages.
2. Nearly identical messages with one subtle difference.
3. Response-time differences.
4. Account locking that affects only existing users.
```

Enumeration reduces the search space:

```text
100 usernames × 100 passwords = 10,000 combinations
100 usernames + 100 passwords = 200 requests
```

Once a valid username is found, it remains fixed while Intruder tests only the password list.

---

### 3. Subtle Response Differences

The only difference may be one character:

```text
Invalid username or password.
Invalid username or password 
```

This is easy to miss visually. Useful techniques include:

- Grep - Match;
- Grep - Extract;
- sorting by Length;
- comparing Status;
- extracting a stable HTML fragment;
- identifying outliers from the dominant response group.

One space, period, tag, or additional HTML branch may become a complete side channel.

---

### 4. Timing Side Channels

Even when response bodies are identical, request processing may differ:

```text
Non-existing username
        ↓
Immediate rejection

Existing username
        ↓
Password-hash computation
        ↓
Slower response
```

Reliable timing analysis requires:

- repeated measurements;
- awareness of network noise;
- controlled concurrency;
- a long password to amplify the difference;
- changing `X-Forwarded-For` when lockout is IP-based;
- identifying a stable latency pattern rather than one isolated outlier.

---

### 5. Brute-Force Protection

A brute-force defense can contain a logic flaw.

Vulnerable pattern:

```text
Failed attempt increments the counter
        ↓
Successful login to the attacker's account resets the counter
        ↓
Requests are interleaved
        ↓
Lockout never occurs
```

Another pattern:

```text
Only an existing account becomes locked
        ↓
The lock message confirms a valid username
```

Rate limiting must not depend on data fully controlled by the attacker and must not reveal account existence.

---

### 6. Burp Intruder Strategy

Different attack models require different Intruder attack types.

**Sniper** is appropriate when one position changes:

```text
password=§candidate§
mfa-code=§0000§
```

**Pitchfork** takes values from multiple payload sets synchronously by index. It can generate an interleaved sequence such as:

```text
wiener:peter
carlos:candidate1
wiener:peter
carlos:candidate2
```

**Cluster Bomb** tests the Cartesian product of payload sets. Combined with Null Payloads, it can repeat each username enough times to trigger account lockout.

---

### 7. Multi-Factor Authentication

An MFA page does not prove that MFA is enforced.

Vulnerable flow:

```text
Password accepted
        ↓
Fully authenticated session created
        ↓
/login2 page displayed
        ↓
/my-account already accessible
```

Secure flow:

```text
Password accepted
        ↓
Restricted pre-auth session created
        ↓
MFA code accepted
        ↓
Session upgraded to authenticated
        ↓
Protected endpoints become accessible
```

The server must verify MFA completion for every protected request.

---

### 8. Broken 2FA Logic

In the vulnerable implementation, the client chooses the account being verified:

```http
Cookie: verify=wiener
```

After tampering:

```http
Cookie: verify=carlos
```

the server generates and validates a code for Carlos even though Carlos's password was never submitted.

Secure invariant:

```text
The factor-two identity must be derived only
from the server-side pre-auth session created by factor one.
```

Never trust a username from a cookie, hidden input, URL, or POST body as the MFA target.

---

### 9. MFA Code Brute Force

A four-digit code has a limited search space:

```text
0000–9999
```

Intruder configuration:

```text
Payload type: Numbers
From: 0
To: 9999
Min integer digits: 4
```

Defensive controls should include:

- short expiration;
- attempt limits;
- lockout or progressive delay;
- single-use semantics;
- invalidating the previous code when issuing a new one;
- binding the code to one pre-auth session.

---

### 10. Persistent Authentication Cookies

A `Stay logged in` option creates a long-lived authentication mechanism.

Vulnerable formula:

```text
Base64(username:MD5(password))
```

Problems:

- Base64 is reversible;
- the username is known;
- MD5 is fast;
- the algorithm is deterministic;
- a candidate cookie can be generated locally for every password;
- the password hash is effectively exposed to the client.

A secure design uses a random opaque token that is revocable and independent of the password.

---

### 11. Online and Offline Cracking

Online brute force:

```text
candidate → HTTP request → response
```

It is constrained by rate limiting, network delay, logging, and detection.

Offline cracking:

```text
stolen hash → local dictionary testing → plaintext
```

Once the hash is obtained, the server is no longer involved in each guess.

Lab chain:

```text
Stored XSS
        ↓
Steal stay-logged-in cookie
        ↓
Base64 decode
        ↓
username:MD5(password)
        ↓
Offline cracking
        ↓
Authenticate as the victim
```

---

### 12. Password Reset Logic

Password reset is an alternative authentication mechanism.

Vulnerable logic:

```text
Token checked separately
Username taken from a hidden field
Server resets the submitted username
```

Tampering:

```http
username=wiener
```

to:

```http
username=carlos
```

Secure model:

```text
reset_token → server-side record → user_id → reset that password
```

A client-supplied username must not determine the target account.

---

### 13. Password Reset Poisoning

Middleware may trust a proxy header:

```http
X-Forwarded-Host: attacker.example
```

and build:

```text
https://attacker.example/forgot-password?temp-forgot-password-token=...
```

When the victim opens the link, the token reaches the exploit server.

Complete chain:

```text
Reset request with X-Forwarded-Host
        ↓
Email contains attacker-controlled domain
        ↓
Victim follows the link
        ↓
Token appears in access log
        ↓
Attacker resets the victim's password
```

---

## 🧩 Authentication Attack Techniques

- **Username Enumeration by Response** — identifying users through text, status, length, or HTML differences.
- **Grep - Extract Analysis** — automatically isolating subtle response variations.
- **Timing Enumeration** — identifying existing users through response latency.
- **IP Lockout Bypass** — abusing trusted proxy headers to evade IP-based controls.
- **Counter Reset Abuse** — resetting a brute-force counter with a valid login.
- **Account Lock Oracle** — treating a lockout message as evidence that an account exists.
- **Password Brute Force** — testing passwords for an already identified username.
- **MFA Step Skipping** — directly accessing protected content before MFA completion.
- **MFA Identity Confusion** — changing the account for which an MFA code is generated or verified.
- **MFA Code Brute Force** — testing a small numerical one-time-code space.
- **Persistent Cookie Forgery** — reproducing a weak remember-me cookie algorithm.
- **Offline Hash Cracking** — recovering a password locally from a stolen fast hash.
- **Password Reset Parameter Tampering** — changing the target username.
- **Host Header Injection** — influencing absolute URLs through host-related headers.
- **Password Reset Poisoning** — redirecting a reset token to an attacker-controlled domain.
- **Session State Analysis** — comparing anonymous, pre-auth, MFA-pending, and authenticated states.

---

## 🧭 Attack Surfaces

| Surface | What to test |
|---------|--------------|
| `POST /login` | messages, Status, Length, timing, redirects, cookies |
| `username` | enumeration, case, whitespace, Unicode, normalization |
| `password` | brute force, length, truncation, timing, policy |
| `X-Forwarded-For` | IP rate-limit bypass and proxy-header trust |
| `POST /login2` | factor-one binding, attempt limits, code reuse |
| `verify` | ability to select another account for MFA |
| `session` | point of full authentication, rotation, logout invalidation |
| `stay-logged-in` | structure, entropy, signature, lifetime, revocation |
| `forgot-password` | enumeration, token generation, expiry, one-time use |
| hidden `username` | trust in a client-controlled field |
| `Host` / `X-Forwarded-Host` | absolute URL construction |
| reset token | entropy, user binding, expiry, replay |

---

## 🛠 Tools

- **Burp Proxy** — intercepting login, MFA, cookie, and password-reset requests.
- **Burp Repeater** — manually testing logic, headers, cookies, and parameters.
- **Burp Intruder** — enumeration, password brute force, and MFA-code guessing.
- **Grep - Match / Grep - Extract** — identifying response differences.
- **Payload Processing** — MD5, prefix/suffix, Base64, and URL encoding.
- **Resource Pool** — controlling concurrency and request order.
- **Burp Decoder** — Base64 decoding and persistent-cookie analysis.
- **Exploit Server** — receiving stolen tokens and XSS callbacks.
- **Email Client** — examining reset links.
- **Browser Developer Tools** — cookies, redirects, and storage.
- **Python / shell utilities** — locally validating formats and hashes in authorized labs.

---

## 📝 Walkthrough Format

Each walkthrough explains not only **what to send**, but also **why the server accepts it**.

Most reports include:

- lab goal;
- short theory;
- correct and vulnerable flows;
- step-by-step exploitation;
- real HTTP requests;
- Burp Intruder configuration;
- payloads used;
- Status, Length, timing, and redirect analysis;
- backend-logic breakdown;
- pentester mindset;
- common mistakes;
- mitigation recommendations;
- final checklist;
- hidden credentials using `<details>`.

---

## 📂 Directory Structure

```text
authentication/
│
├── README.md
├── README_EN.md
│
├── 001_ru_username_enumeration_via_different_responses.md
├── 001_en_username_enumeration_via_different_responses.md
├── 002_ru_username_enumeration_via_subtly_different_responses.md
├── 002_en_username_enumeration_via_subtly_different_responses.md
├── 003_ru_username_enumeration_via_response_timing.md
├── 003_en_username_enumeration_via_response_timing.md
├── 004_ru_broken_bruteforce_protection_ip_block.md
├── 004_en_broken_bruteforce_protection_ip_block.md
├── 005_ru_username_enumeration_via_account_lock.md
├── 005_en_username_enumeration_via_account_lock.md
├── 006_ru_2fa_simple_bypass.md
├── 006_en_2fa_simple_bypass.md
├── 007_ru_2fa_broken_logic.md
├── 007_en_2fa_broken_logic.md
├── 008_ru_brute_forcing_stay_logged_in_cookie.md
├── 008_en_brute_forcing_stay_logged_in_cookie.md
├── 009_ru_offline_password_cracking.md
├── 009_en_offline_password_cracking.md
├── 010_ru_password_reset_broken_logic.md
├── 010_en_password_reset_broken_logic.md
├── 011_ru_password_reset_poisoning_via_middleware.md
└── 011_en_password_reset_poisoning_via_middleware.md
```

---

## 🎓 What You Will Learn

After completing this section, you will be able to:

- test login flows systematically;
- identify username enumeration through text, length, status, timing, and account locking;
- choose Sniper, Pitchfork, or Cluster Bomb for the attack model;
- use Grep - Match and Grep - Extract;
- analyze rate limiting and account-lock logic;
- identify IP-lockout bypasses and counter-reset flaws;
- verify server-side enforcement of the second factor;
- identify MFA identity confusion;
- configure numerical code brute force with leading zeros;
- reverse-engineer persistent authentication cookies;
- distinguish Base64 from encryption and signing;
- explain MD5 and offline-cracking risks;
- analyze reset-token user binding;
- test Host Header Injection and reset poisoning;
- explain vulnerabilities as backend state-machine failures;
- recommend practical defensive controls.

---

## 🛡️ Authentication Defense

### Uniform Responses

For a non-existing username and an incorrect password, the following should be equivalent:

```text
HTTP status
Response body
Response length
Response time
Redirect behavior
```

Identical wording is not enough if timing or HTML structure still differs.

### Rate Limiting and Account Lockout

- apply limits per account and IP;
- use progressive delays;
- incorporate additional device and session signals;
- monitor credential stuffing;
- provide a secure unlock process;
- notify the account owner;
- trust proxy headers only from a known reverse proxy;
- eliminate logic paths that reset the counter unexpectedly.

### Secure MFA

- create a restricted pre-auth session after password verification;
- keep the target user ID server-side;
- never accept the MFA identity from a cookie or request parameter;
- check MFA state on every protected endpoint;
- use short-lived, single-use codes;
- limit verification attempts;
- invalidate previous codes;
- rotate the session after MFA completion.

### Persistent Login

- use random opaque tokens;
- store a server-side hash of the token;
- never place a password hash in the cookie;
- enforce expiration and per-device revocation;
- rotate tokens after use;
- revoke tokens after password changes;
- use `Secure`, `HttpOnly`, and an appropriate `SameSite`.

### Password Storage

Use:

```text
Argon2id
scrypt
bcrypt
PBKDF2
```

Do not use:

```text
MD5
SHA-1
plain SHA-256 without salt and work factor
```

### Password Reset

- generate a high-entropy random token;
- store only a server-side token hash;
- bind `token → user_id`;
- enforce single use and short expiry;
- invalidate previous tokens;
- build URLs from a configured trusted base URL;
- do not trust external `Host` or `X-Forwarded-Host` values;
- revoke sessions and persistent tokens after reset;
- return the same response for existing and non-existing accounts.

---

## 🔗 Useful Links

- PortSwigger Authentication: https://portswigger.net/web-security/authentication
- Password-Based Authentication: https://portswigger.net/web-security/authentication/password-based
- Multi-Factor Authentication: https://portswigger.net/web-security/authentication/multi-factor
- Other Authentication Mechanisms: https://portswigger.net/web-security/authentication/other-mechanisms
- Burp Intruder: https://portswigger.net/burp/documentation/desktop/tools/intruder
- Web Security Academy: https://portswigger.net/web-security

---

## 🚀 Next Steps

After Authentication, logical next sections are:

- Access Control;
- OAuth Authentication;
- JWT;
- Business Logic Vulnerabilities;
- Cross-Site Scripting;
- CSRF;
- Web Cache Poisoning;
- Server-Side Vulnerabilities.

---

⬆ [Back to top](#-authentication--portswigger-web-security-academy)
