# 📘 PortSwigger Lab: Exploiting path delimiters for web cache deception

> 🔗 **Open the lab:**  
> https://portswigger.net/web-security/web-cache-deception/lab-wcd-exploiting-path-delimiters

---

## 📑 Table of Contents

- [🎯 Goal](#-goal)
- [🧠 Key Idea](#-key-idea)
- [🔍 Solution Walkthrough](#-solution-walkthrough)
- [💥 Vulnerability](#-vulnerability)
- [🛡️ Recommendations](#️-recommendations)
- [🔐 Solution](#-solution)
- [🛠️ Checklist](#️-checklist)

---

## 🎯 Goal

Retrieve the API key for user `carlos` by exploiting a **Web Cache Deception (WCD)** issue caused by discrepancies in how the origin server and the cache interpret **path delimiters**.

---

## 🧠 Key Idea

In this lab, the vulnerability is not caused by ordinary path mapping. Instead, it comes from the fact that the **origin server and the cache interpret special URL characters differently**.

In our case, we were interested in symbols that could act as delimiters. The attack works as follows:

- the **origin server** treats a specific symbol as a delimiter and effectively processes the request as `/my-account`;
- the **cache** may not treat the same symbol as a delimiter and may continue analyzing the rest of the path;
- if the remaining part looks like `.js`, the cache may classify it as a **static resource** and store the response;
- because the origin still returns a **personalized account page**, another user's sensitive response ends up in the cache.

The core formula for this lab is:

> **Origin thinks:** “this is `/my-account`”  
> **Cache thinks:** “this is a `.js` resource that can be cached”

---

## 🔍 Solution Walkthrough

### Step 1 — Identify the target endpoint

Log in with:

```text
wiener:peter
```

After login, navigate to the account page and observe:

- username;
- API key;
- CSRF token.

Target endpoint:

```text
/my-account
```

This is clearly a sensitive endpoint because it returns **user-specific data**.

---

### Step 2 — Confirm this is not the previous path-mapping case

First, it is useful to verify that this lab does not behave like the previous one, where the origin ignored additional path segments.

Try:

```http
GET /my-account/abc HTTP/2
```

Observation from the official solution:

- the server returns `404 Not Found`;
- there is no evidence of caching.

Conclusion:

- the origin does **not** abstract `/my-account/abc` to `/my-account`;
- therefore this is **not** the same path mapping discrepancy as in the previous lab.

Next, send another baseline request:

```http
GET /my-accountabc HTTP/2
```

Observation:

- again, `404 Not Found`;
- no evidence of caching.

This step matters as a **baseline**: it shows what a normal invalid path looks like.

---

### Step 3 — Identify delimiters used by the origin

The lab provides a list of candidate delimiter characters.  
This is not just background information. It is intended to be used for **systematic fuzzing**.

The official approach is:

1. Send the baseline request to **Intruder**.
2. Place the payload position immediately after `/my-account`:

```text
/my-account§§abc
```

3. Load the candidate delimiter list as payloads.
4. Disable automatic URL-encoding for payload characters.
5. Run the attack and sort the results by `Status code`.

Official result:

- `;` → `200 OK`
- `?` → `200 OK`
- all other characters → `404 Not Found`

Conclusion:

- the **origin server uses `;` and `?` as delimiters**
- requests such as:

```text
/my-account;abc
/my-account?abc
```

still resolve to the account page.

---

### Step 4 — Manual verification of the working delimiter

In our hands-on approach, we did not fuzz the entire list with Intruder first.  
Instead, we manually tested `;` straight away.

Request:

```http
GET /my-account;test HTTP/2
```

Observation:

- no error is returned;
- the server responds with our account page and our API key.

Conclusion:

- the **origin ignores everything after `;`**;
- for the origin, this URL is still effectively `/my-account`.

This already gave us a usable delimiter candidate.

---

### Step 5 — Test caching with a static-looking extension

The next step is to determine whether the cache will treat this URL as a cacheable static resource.

Try:

```http
GET /my-account;test.js HTTP/2
```

Observation:

- the response contains `X-Cache: miss`;
- on a repeated request, this changes to `X-Cache: hit`.

This means:

- the cache does **not** treat `;` as a delimiter in the same way as the origin;
- the cache analyzes the full path;
- because of `.js`, it applies a static-file caching rule.

This is the key discrepancy.

---

### Step 6 — Why comparing `;` and `?` matters

The official solution deliberately compares two symbols that both produce `200 OK` at the origin layer:

- `?`
- `;`

Testing:

```text
/my-account?abc.js
```

does **not** produce evidence of caching.

Testing:

```text
/my-account;abc.js
```

does produce `X-Cache: miss`, then `hit`.

Why?

- **`?` is a delimiter for both the origin and the cache**
- **`;` is a delimiter for the origin, but not for the cache**

This leads to the main conclusion of the lab:

> For successful WCD exploitation, it is not enough to find a delimiter.  
> You need a delimiter that the **origin understands**, but the **cache does not**.

This was an important methodological point that we did not initially verify in our manual approach, but it became clear after comparing our work with the official solution.

---

### Step 7 — Prepare the exploit

Once a working delimiter is identified, the remaining step is to make victim user `carlos` request a **new URL** that we have not used ourselves.

We chose:

```text
/my-account;bug.js
```

Why this matters:

- the path must be **new**, otherwise the cache may already contain our own response;
- whoever makes the **first request** to a fresh cache key determines what gets stored.

On the exploit server, use:

```html
<script>document.location="https://YOUR-LAB-ID.web-security-academy.net/my-account;bug.js"</script>
```

---

### Step 8 — Deliver the exploit to the victim

On the exploit server:

1. Click `Store`
2. Click `Deliver exploit to victim`

After that, the victim browser navigates to:

```text
/my-account;bug.js
```

What happens next:

1. The **origin** treats `;bug.js` as data after a delimiter and still returns the `/my-account` page.
2. The **cache** treats the same URL as a static-looking `.js` path and stores the response.
3. The cache now contains **Carlos's account data**, not ours.

---

### Step 9 — Retrieve the result

After the exploit has been delivered, open the same URL in the browser:

```text
https://YOUR-LAB-ID.web-security-academy.net/my-account;bug.js
```

Result:

- the response shows **Carlos's API key**;
- the lab is solved.

---

## 💥 Vulnerability

The vulnerability is caused by a **discrepancy between the origin server and the cache in how they interpret path delimiters**.

In this lab specifically:

- `;` is a delimiter for the **origin**;
- `;` is **not** treated as a delimiter by the **cache**;
- therefore, the origin returns a **dynamic, personalized response** for `/my-account`;
- while the cache classifies the same URL as a **cacheable static resource** because of `.js`.

This allows sensitive data from another user to be stored in a shared cache and exposed to an attacker.

### What matters methodologically

Comparing our approach with the official solution highlights the following:

- we found a working delimiter manually and exploited the issue successfully;
- the official solution also demonstrates the **systematic approach**:
  - establish a baseline first;
  - fuzz the delimiter list;
  - compare symbols that work at the origin layer but behave differently at the cache layer.

So the delimiter list in the lab is not merely informational. It is meant to be used as a practical input for discovering discrepancies.

---

## 🛡️ Recommendations

- **Do not cache sensitive endpoints**  
  For pages such as `/my-account`, `/profile`, `/settings`, `/api/me`, use headers like:
  ```http
  Cache-Control: no-store, no-cache, private
  ```

- **Enforce strict routing**  
  If a request does not exactly match `/my-account`, the server should return `404` or `403`, rather than accepting suffixes such as `;...` or `?...` as equivalent paths.

- **Align cache and origin parsing logic**  
  Cache and origin should interpret the following in the same way:
  - delimiters;
  - encoded characters;
  - query/path boundaries;
  - unusual URL segments.

- **Do not rely only on file extensions**  
  A response should not be cached simply because the URL ends in `.js`, `.css`, `.png`, and so on.  
  Caching decisions should also consider:
  - cookies;
  - authentication state;
  - endpoint type;
  - whether the response is personalized.

- **Use `Vary: Cookie` and private caching semantics where appropriate**  
  This reduces the risk of leaking user-specific responses through shared caches.

- **Fuzz delimiters and encoded delimiters during testing**  
  It is valuable to test both literal characters (`;`, `?`, `/`, `.`) and URL-encoded versions (`%3B`, `%3F`, `%2F`, etc.), because different layers may interpret them differently.

---

## 🔐 Solution

<details>
<summary>Show solution</summary>

```text
the key is individual for each session
```

</details>

---

## 🛠️ Checklist

- [ ] A sensitive endpoint containing personal data was identified
- [ ] Baseline behavior for invalid paths was verified (`404` / no caching)
- [ ] Delimiters understood by the origin were identified
- [ ] It was checked which delimiters are / are not also understood by the cache
- [ ] A URL was found that the origin treats as the account page while the cache treats it as static content
- [ ] A fresh cache key was used for the victim
- [ ] The victim made the first request to that URL
- [ ] The cached response exposed Carlos's data
