# 📘 PortSwigger Lab: Broken Brute-force Protection, IP Block

<a id="top"></a>

> 🔗 **Official Lab:** https://portswigger.net/web-security/authentication/password-based/lab-broken-bruteforce-protection-ip-block  
> 📄 **Candidate passwords:** https://portswigger.net/web-security/authentication/auth-lab-passwords  
> 🎯 **Topic:** Authentication vulnerabilities — Flawed Brute-force Protection / IP Block Logic Flaw  
> 🧪 **Difficulty:** Practitioner  
> 👤 **Own account:** `wiener:peter`  
> 🎯 **Target user:** `carlos`

---

## 📑 Contents

- [🎯 Goal](#goal)
- [🧠 Short Theory](#theory)
- [🔐 Password-based Authentication](#password-auth)
- [💣 Brute Force, Credential Stuffing, and Password Spraying](#bruteforce-types)
- [🛡 Common Brute-force Protections](#protection)
- [🧩 Core Idea](#idea)
- [💥 The Logic Flaw](#logic-flaw)
- [🧪 Why This Lab Matters](#importance)
- [🎯 Why Pitchfork Is Used](#pitchfork)
- [🚫 Why Not Sniper](#not-sniper)
- [🚫 Why Not Cluster Bomb](#not-cluster-bomb)
- [⚙ Why Resource Pool = 1 Is Required](#resource-pool)
- [🔍 Step 1 — Inspect the Login Page](#step1)
- [🔍 Step 2 — Confirm the Lockout Behavior](#step2)
- [🔍 Step 3 — Confirm the Counter Reset with Your Own Account](#step3)
- [🔍 Step 4 — Send the Request to Intruder](#step4)
- [🔍 Step 5 — Configure Pitchfork](#step5)
- [🔍 Step 6 — Prepare the Username Payload List](#step6)
- [🔍 Step 7 — Prepare the Password Payload List](#step7)
- [🔍 Step 8 — Configure the Resource Pool](#step8)
- [🔍 Step 9 — Start the Attack](#step9)
- [🔍 Step 10 — Identify Carlos's Password](#step10)
- [🔍 Step 11 — Log in to the Account](#step11)
- [🧩 Payloads Used](#payloads)
- [🔬 Why the Attack Worked](#breakdown)
- [🌍 Real-World Relevance](#real-world)
- [🧠 How to Think Like a Pentester](#pentester)
- [❌ Common Mistakes](#mistakes)
- [🛡 Security Recommendations](#defense)
- [🎓 Key Takeaways](#takeaways)
- [✅ Checklist](#checklist)

---

<a id="goal"></a>

## 🎯 Goal

The goal of this lab is to brute-force the password for the user:

```text
carlos
```

and access his account page.

Unlike the previous username enumeration labs, the victim's username is already known. The main task is to understand why the brute-force protection exists but can still be bypassed due to flawed application logic.

The lab provides your own valid credentials:

```text
wiener:peter
```

These credentials are not the final target. They are used as a tool to bypass the brute-force protection.

[⬆ Back to contents](#top)

---

<a id="theory"></a>

## 🧠 Short Theory

**Flawed brute-force protection** means that an application attempts to prevent password guessing, but the implementation contains a logic flaw that allows the protection to be bypassed.

This is important because the vulnerability is not caused by the complete absence of security controls. The protection exists, but its logic is broken.

In this lab, the application blocks the IP address after several failed login attempts. At first glance, this looks like reasonable brute-force protection. However, if the attacker successfully logs in to their own account before the limit is reached, the failed-attempt counter is reset.

This makes it possible to alternate between:

```text
a successful login using the attacker's own account
        ↓
one password guess for Carlos
        ↓
another successful login using the attacker's own account
        ↓
another password guess for Carlos
```

As a result, the lockout threshold is never reached.

[⬆ Back to contents](#top)

---

<a id="password-auth"></a>

## 🔐 Password-based Authentication

Password-based authentication is the classic login mechanism based on:

```text
username + password
```

A typical application performs the following flow:

```text
1. Receive username and password.
2. Look up the user in the database.
3. Verify the password.
4. Create a session on successful login.
5. Return an error on failed login.
```

Simplified flow:

```text
POST /login
    ↓
Is username valid?
    ↓
Is password valid?
    ↓
Create session cookie
    ↓
Redirect to /my-account
```

When the password is incorrect, the application should return an error and record the failed attempt.

The security issue starts when developers make incorrect assumptions about:

```text
where the failed-attempt counter is stored
when the counter should be reset
which user the counter belongs to
whether a successful login should reset all failures
```

[⬆ Back to contents](#top)

---

<a id="bruteforce-types"></a>

## 💣 Brute Force, Credential Stuffing, and Password Spraying

### Brute Force

A brute-force attack tries many passwords against one account:

```text
carlos:123456
carlos:password
carlos:qwerty
carlos:football
```

### Credential Stuffing

Credential stuffing uses leaked username/password pairs from other services:

```text
user@example.com:Password123
admin@example.com:qwerty2024
```

The attacker does not necessarily guess passwords. They reuse known credentials.

### Password Spraying

Password spraying tries one common password against many accounts:

```text
alice:Winter2024
bob:Winter2024
carlos:Winter2024
admin:Winter2024
```

This lab is closest to a single-account brute-force attack, but with a logic flaw that allows the attacker to bypass IP-based protection.

[⬆ Back to contents](#top)

---

<a id="protection"></a>

## 🛡 Common Brute-force Protections

Common brute-force protections include:

```text
1. IP-based blocking
2. Account lockout
3. Rate limiting
4. Progressive delay
5. CAPTCHA
6. MFA
7. Monitoring and alerting
```

### IP-based Blocking

```text
3 failed attempts from one IP
        ↓
temporary IP block
```

### Account Lockout

```text
5 failed attempts for carlos
        ↓
temporarily lock Carlos's account
```

### Progressive Delay

```text
1st failure  → 1 second delay
2nd failure  → 5 second delay
3rd failure  → 30 second delay
```

### MFA

Even if the password is guessed, the attacker still needs a second factor.

In this lab, the application uses IP-based blocking, but the counter-reset logic is flawed.

[⬆ Back to contents](#top)

---

<a id="idea"></a>

## 🧩 Core Idea

A normal brute-force attempt gets blocked:

```text
carlos:123456    ❌
carlos:password  ❌
carlos:qwerty    ❌
        ↓
BLOCK
```

However, if we insert a successful login using our own account between attempts, the counter is reset:

```text
wiener:peter     ✅
carlos:123456    ❌
wiener:peter     ✅
carlos:password  ❌
wiener:peter     ✅
carlos:qwerty    ❌
```

The logic becomes:

```text
Successful login
      ↓
failed_attempts = 0
      ↓
Wrong Carlos password
      ↓
failed_attempts = 1
      ↓
Successful login
      ↓
failed_attempts = 0
```

Main idea:

```text
We do not remove the lockout.
We abuse the application's own reset logic.
```

[⬆ Back to contents](#top)

---

<a id="logic-flaw"></a>

## 💥 The Logic Flaw

A vulnerable implementation may look like this:

```python
if login_failed:
    failed_attempts += 1

if login_success:
    failed_attempts = 0

if failed_attempts >= 3:
    block_ip()
```

At first glance, this may look reasonable:

```text
if the user successfully logs in,
they are probably legitimate,
so the failed-attempt counter can be reset
```

The problem is that the successful login may belong to a completely different user.

In this lab:

```text
The failed attempts target Carlos.
The reset is performed by Wiener.
```

A correct implementation should not allow one user's successful login to reset the brute-force state for another user's account.

Weak model:

```text
failed_attempts[ip]
```

Better models:

```text
failed_attempts[ip + username]
failed_attempts[username]
failed_attempts[ip]
failed_attempts[device/session]
```

The core issue is that the application resets too much security state after a successful login.

[⬆ Back to contents](#top)

---

<a id="importance"></a>

## 🧪 Why This Lab Matters

This lab teaches an important lesson:

> The presence of a security control does not mean the control is correctly implemented.

During a real penetration test, it is not enough to observe that lockout exists and stop there. You must understand how it works.

Important questions:

```text
When does the lockout trigger?
What counts as a failed attempt?
What resets the counter?
Is the counter global?
Is the counter tied to an IP?
Is it tied to a username?
Can my own valid account influence the counter?
```

These issues are often missed by automated scanners because they require business-logic analysis.

[⬆ Back to contents](#top)

---

<a id="pitchfork"></a>

## 🎯 Why Pitchfork Is Used

Burp Intruder provides different attack types. In this lab, **Pitchfork** is the correct choice.

We need two payload lists to move in parallel:

| Request | Username | Password |
|---:|---|---|
| 1 | wiener | peter |
| 2 | carlos | 123456 |
| 3 | wiener | peter |
| 4 | carlos | password |
| 5 | wiener | peter |
| 6 | carlos | qwerty |

Pitchfork takes values line by line:

```text
payload1[1] + payload2[1]
payload1[2] + payload2[2]
payload1[3] + payload2[3]
```

This preserves the required pairing.

[⬆ Back to contents](#top)

---

<a id="not-sniper"></a>

## 🚫 Why Not Sniper

Sniper changes only one payload position.

For example:

```http
username=carlos&password=§password§
```

This results in a normal brute-force sequence:

```text
carlos:123456
carlos:password
carlos:qwerty
```

After several failures, the IP is blocked.

Sniper is not suitable because both `username` and `password` must change in a synchronized way.

[⬆ Back to contents](#top)

---

<a id="not-cluster-bomb"></a>

## 🚫 Why Not Cluster Bomb

Cluster Bomb tries all combinations:

```text
wiener:peter
wiener:123456
wiener:password
carlos:peter
carlos:123456
carlos:password
```

This breaks the required order.

We do not need every possible combination. We need a strict alternating pattern:

```text
wiener:peter
carlos:candidate
wiener:peter
carlos:candidate
```

Therefore, Cluster Bomb is not appropriate for this lab.

[⬆ Back to contents](#top)

---

<a id="resource-pool"></a>

## ⚙ Why Resource Pool = 1 Is Required

The order of requests is critical.

If Burp sends multiple requests in parallel, the server may process them in the wrong order.

Expected order:

```text
1. wiener:peter
2. carlos:123456
3. wiener:peter
4. carlos:password
```

Bad order caused by parallelism:

```text
2. carlos:123456
4. carlos:password
6. carlos:qwerty
1. wiener:peter
```

In this case, multiple Carlos failures may arrive consecutively and trigger the lockout.

The correct configuration is:

```text
Resource Pool
Maximum concurrent requests = 1
```

This forces Burp to send one request at a time.

[⬆ Back to contents](#top)

---

<a id="step1"></a>

## 🔍 Step 1 — Inspect the Login Page

Open the login page and submit test credentials:

```text
username=test
password=test
```

In Burp Proxy → HTTP history, find:

```http
POST /login
```

Example request body:

```http
username=test&password=test
```

This request will be used as the base for Intruder.

[⬆ Back to contents](#top)

---

<a id="step2"></a>

## 🔍 Step 2 — Confirm the Lockout Behavior

Test how the protection works.

Submit several invalid attempts:

```text
carlos:test1
carlos:test2
carlos:test3
```

After the threshold is reached, the application blocks login attempts from the IP address.

Conclusion:

```text
Brute-force protection exists.
Now we need to test its logic.
```

[⬆ Back to contents](#top)

---

<a id="step3"></a>

## 🔍 Step 3 — Confirm the Counter Reset with Your Own Account

Test this sequence:

```text
carlos:test1  ❌
carlos:test2  ❌
wiener:peter  ✅
carlos:test3  ❌
```

If no lockout occurs after the final request, it means that a successful login using `wiener:peter` resets the failed-attempt counter.

This is the vulnerability.

[⬆ Back to contents](#top)

---

<a id="step4"></a>

## 🔍 Step 4 — Send the Request to Intruder

Send the `POST /login` request to Intruder.

Set two payload positions:

```http
username=§wiener§&password=§peter§
```

Two positions are required because username and password must change in parallel.

[⬆ Back to contents](#top)

---

<a id="step5"></a>

## 🔍 Step 5 — Configure Pitchfork

Select:

```text
Attack type: Pitchfork
```

This allows Burp to use two payload lists line by line.

[⬆ Back to contents](#top)

---

<a id="step6"></a>

## 🔍 Step 6 — Prepare the Username Payload List

Username list:

```text
wiener
carlos
wiener
carlos
wiener
carlos
...
```

`wiener` must come first because the first entry in the password payload list is `peter`.

The first request must be successful:

```text
wiener:peter
```

[⬆ Back to contents](#top)

---

<a id="step7"></a>

## 🔍 Step 7 — Prepare the Password Payload List

Password list:

```text
peter
123456
peter
password
peter
qwerty
peter
football
...
```

Each `wiener` entry is paired with `peter`, and each `carlos` entry is paired with the next candidate password.

Important:

```text
The number of rows in both payload lists must match.
```

[⬆ Back to contents](#top)

---

<a id="step8"></a>

## 🔍 Step 8 — Configure the Resource Pool

In Intruder, configure the Resource Pool:

```text
Maximum concurrent requests = 1
```

This ensures that requests are not reordered.

[⬆ Back to contents](#top)

---

<a id="step9"></a>

## 🔍 Step 9 — Start the Attack

Start the Intruder attack.

Most responses are expected to be:

```http
200 OK
```

A successful login is identified by:

```http
302 Found
```

[⬆ Back to contents](#top)

---

<a id="step10"></a>

## 🔍 Step 10 — Identify Carlos's Password

In the Intruder results, find the request where:

```text
username=carlos
status=302
```

This means that the password was correct.

The discovered password is hidden:

<details>
<summary>🔑 Show discovered password</summary>

```text
klaster
```

</details>

[⬆ Back to contents](#top)

---

<a id="step11"></a>

## 🔍 Step 11 — Log in to the Account

Log in with:

```text
username=carlos
password=<discovered_password>
```

Credentials are hidden:

<details>
<summary>🔑 Show discovered credentials</summary>

```text
Username: carlos
Password: klaster
```

</details>

After accessing Carlos's account page, the lab is solved.

[⬆ Back to contents](#top)

---

<a id="payloads"></a>

## 🧩 Payloads Used

Username pattern:

```text
wiener
carlos
wiener
carlos
```

Password pattern:

```text
peter
candidate_password_1
peter
candidate_password_2
```

Full logic:

```text
wiener:peter              ✅ reset
carlos:candidate_1        ❌ test
wiener:peter              ✅ reset
carlos:candidate_2        ❌ test
```

[⬆ Back to contents](#top)

---

<a id="breakdown"></a>

## 🔬 Why the Attack Worked

The attack worked because two conditions were true:

1. The application blocked the IP after several failures.
2. A successful login by another user reset the failed-attempt counter.

This allowed the attacker to:

```text
make one guess against Carlos
        ↓
reset the counter using Wiener
        ↓
make the next guess against Carlos
```

This is not a direct bypass of the lockout mechanism. It is an abuse of the application's state-management logic.

[⬆ Back to contents](#top)

---

<a id="real-world"></a>

## 🌍 Real-World Relevance

This type of vulnerability appears in real applications when:

- rate limiting is implemented manually;
- the state is stored only per IP address;
- the failed-attempt counter is global;
- successful login clears too much security state;
- there is no per-user lockout logic;
- developers do not test account alternation patterns.

In bug bounty and pentest engagements, this can be impactful because it may lead to account takeover.

[⬆ Back to contents](#top)

---

<a id="pentester"></a>

## 🧠 How to Think Like a Pentester

Correct reasoning flow:

```text
Is there brute-force protection?
    ↓
Yes
    ↓
When does it trigger?
    ↓
After several failures
    ↓
What resets the counter?
    ↓
A successful login
    ↓
Can I perform a successful login?
    ↓
Yes, I have wiener:peter
    ↓
Can I alternate requests?
    ↓
Yes
    ↓
Automate with Pitchfork
```

Main lesson:

```text
A pentester does not only check whether protection exists.
A pentester checks whether the protection logic is correct.
```

[⬆ Back to contents](#top)

---

<a id="mistakes"></a>

## ❌ Common Mistakes

### 1. Using Sniper

Sniper cannot alternate username and password lists in parallel.

### 2. Using Cluster Bomb

Cluster Bomb creates all combinations and breaks the required sequence.

### 3. Not configuring Resource Pool

Parallel requests may trigger the lockout.

### 4. Misaligning payload lists

Payload lists must match row by row.

### 5. Starting with Carlos

The first pair should be `wiener:peter`, so the sequence starts with a successful login.

### 6. Continuing after 302

Once the correct password is found, the attack should be stopped.

[⬆ Back to contents](#top)

---

<a id="defense"></a>

## 🛡 Security Recommendations

Proper protection should include:

- a separate failed-attempt counter per username;
- a separate counter per IP address;
- combined tracking such as `IP + username`;
- progressive delays;
- MFA;
- monitoring;
- detection of login alternation patterns;
- alerts for suspicious login behavior.

Bad design:

```text
any successful login resets the global failed-attempt counter
```

Better design:

```text
a successful login for user A
must not reset failed attempts against user B
```

[⬆ Back to contents](#top)

---

<a id="takeaways"></a>

## 🎓 Key Takeaways

This lab teaches:

- how to analyze brute-force protection logic;
- how to identify counter reset conditions;
- why Pitchfork is useful for paired payloads;
- why request order matters;
- why Resource Pool configuration is important;
- why authentication vulnerabilities are often business-logic vulnerabilities.

[⬆ Back to contents](#top)

---

<a id="checklist"></a>

## ✅ Checklist

- [x] Lockout behavior confirmed.
- [x] Counter reset through `wiener:peter` confirmed.
- [x] `POST /login` request identified.
- [x] Request sent to Intruder.
- [x] Pitchfork selected.
- [x] Username payload list prepared.
- [x] Password payload list prepared.
- [x] Payload lists aligned.
- [x] Resource Pool set to `Maximum concurrent requests = 1`.
- [x] `302 Found` response identified.
- [x] Password hidden in the report.
- [x] Carlos's account accessed.
- [x] Lab solved.

---

# ⬆ Back to Top

[Return to contents](#top)
