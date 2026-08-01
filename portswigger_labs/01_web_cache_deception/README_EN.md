# 🧊 Web Cache Deception — PortSwigger Web Security Academy

<p align="center">
  <img src="https://img.shields.io/badge/Platform-PortSwigger-red" />
  <img src="https://img.shields.io/badge/Topic-Web%20Cache%20Deception-blue" />
  <img src="https://img.shields.io/badge/Labs-4-success" />
  <img src="https://img.shields.io/badge/Progress-100%25-brightgreen" />
  <img src="https://img.shields.io/badge/Language-RU%20%7C%20EN-purple" />
</p>

<p align="right">
🇷🇺 <a href="./README.md"><b>Russian Version</b></a>
</p>

> 📂 Path: `cyber_sec_labs/portswigger_labs/web_cache_deception`  
> 📚 Section: Web Cache Deception  
> 🧪 Labs: 4  
> 📝 Reports: 8 — Russian and English versions for each lab

---

## 📚 Table of Contents

- [About](#-about)
- [Disclaimer](#️-disclaimer)
- [Progress](#-progress)
- [Learning Roadmap](#️-learning-roadmap)
- [Lab Map](#-lab-map)
- [What Is Web Cache Deception](#-what-is-web-cache-deception)
- [Key Concepts](#-key-concepts)
- [Core Techniques](#-core-techniques)
- [Tools](#-tools)
- [Walkthrough Format](#-walkthrough-format)
- [Directory Structure](#-directory-structure)
- [What You Will Learn](#-what-you-will-learn)
- [Web Cache Deception Prevention](#️-web-cache-deception-prevention)
- [Useful Links](#-useful-links)
- [Next Steps](#-next-steps)

---

## 📌 About

This directory contains walkthroughs for the **Web Cache Deception** labs from **PortSwigger Web Security Academy**.

The goal is to understand how discrepancies between the **origin server** and the **cache server** can cause user-specific private data to be stored in a shared cache and later retrieved by an attacker.

In these labs, the main idea is not to break the backend directly. Instead, the attacker makes different infrastructure layers interpret the same URL differently.

---

## ⚠️ Disclaimer

These materials are for **educational purposes only**.

All actions were performed only inside the legal PortSwigger Web Security Academy lab environment. Do not use these techniques against real systems without explicit written authorization.

---

## 📊 Progress

```text
Section: Web Cache Deception
Labs: 4 / 4
Reports: 8
Languages: RU / EN
Status: completed
```

```text
████ 4 / 4 — 100%
```

---

## 🗺️ Learning Roadmap

```text
Web Cache Deception
│
├── Path Mapping
│   └── the origin ignores the path suffix while the cache sees a static resource
│
├── Path Delimiters
│   └── the origin and the cache interpret URL delimiters differently
│
├── Origin Server Normalization
│   └── the origin normalizes the path differently from the cache
│
├── Cache Server Normalization
│   └── the cache normalizes the path differently from the origin
│
└── Defense
    ├── no-store / private
    ├── Vary: Cookie
    ├── strict routing
    ├── consistent normalization
    └── safe cache rules
```

---

## 📋 Lab Map

| # | Lab | What it teaches | RU | EN |
|---|-----|-----------------|:--:|:--:|
| 001 | **Exploiting Path Mapping for Web Cache Deception** | a discrepancy between a dynamic origin endpoint and a static-looking cache URL | [RU](./001_ru_exploiting_path_mapping_for_web_cache_deception.md) | [EN](./001_en_exploiting_path_mapping_for_web_cache_deception.md) |
| 002 | **Exploiting Path Delimiters for Web Cache Deception** | finding path delimiters interpreted differently by the origin and the cache | [RU](./002_ru_exploiting_path_delimiters_for_web_cache_deception.md) | [EN](./002_en_exploiting_path_delimiters_for_web_cache_deception.md) |
| 003 | **Exploiting Origin Server Normalization for Web Cache Deception** | URL normalization differences where the origin normalizes the path but the cache applies caching rules differently | [RU](./003_ru_exploiting_origin_server_normalization_for_web_cache_deception.md) | [EN](./003_en_exploiting_origin_server_normalization_for_web_cache_deception.md) |
| 004 | **Exploiting Cache Server Normalization for Web Cache Deception** | parser discrepancy where the cache normalizes the path differently from the origin and stores a private response | [RU](./004_ru_exploiting_cache_server_normalization_for_web_cache_deception.md) | [EN](./004_en_exploiting_cache_server_normalization_for_web_cache_deception.md) |

---

## 🧠 What Is Web Cache Deception

**Web Cache Deception (WCD)** is a vulnerability where an attacker tricks a cache into storing a private user-specific response as if it were a public static resource.

Simplified flow:

```text
The victim opens a crafted URL
        ↓
The origin server returns a private page
        ↓
The cache treats the URL as a static resource
        ↓
The cache stores the private response
        ↓
The attacker opens the same URL and retrieves the victim's data
```

Main issue:

```text
The origin server and the cache server interpret the URL differently.
```

---

## 🧩 Key Concepts

### Origin server

The main backend server of the application. It understands sessions, cookies, business logic, and private endpoints such as:

```text
/my-account
/profile
/settings
/api/me
```

### Cache server

An intermediate layer such as a CDN, reverse proxy, or caching proxy. It decides whether a response can be stored and reused.

### Cache key

The key used by the cache to store and retrieve a response. It is usually based on the URL and may also include headers, query strings, or cookies.

### Static-looking URL

A URL that looks like a static resource:

```text
/file.js
/image.png
/styles.css
/resources/...
```

If the cache relies too much on file extensions or path prefixes, it may store a private response.

### Parser discrepancy

A mismatch in URL parsing:

```text
The origin sees one thing
The cache sees another thing
```

This is the core idea behind all labs in this section.

### Cache buster

A unique value added to a URL to create a fresh cache key:

```text
?wcd123
/random.js
```

It ensures that the victim is the first user to populate the cache with their own data.

---

## 🧪 Core Techniques

### 1. Path Mapping

The origin treats:

```text
/my-account/abc.js
```

as:

```text
/my-account
```

while the cache sees `.js` and considers the response static.

### 2. Path Delimiters

Some characters may act as delimiters for the origin but not for the cache.

Example:

```text
/my-account;test.js
```

The origin may process this as `/my-account`, while the cache treats it as a `.js` resource.

### 3. Origin Server Normalization

The origin decodes and normalizes the path:

```text
/resources/..%2fmy-account
→ /my-account
```

while the cache still applies a caching rule based on `/resources`.

### 4. Cache Server Normalization

The cache normalizes the path and sees `/resources`, while the origin still returns `/my-account` due to delimiter handling.

Core idea:

```text
Origin → /my-account
Cache  → /resources
```

### 5. Cache Poisoning / Cache Deception Chain

In WCD exploitation, the victim must make the first request to a fresh cache key. This causes the cache to store the victim's private response.

---

## 🛠 Tools

- **Burp Proxy** — intercepting and analyzing HTTP requests.
- **Burp Repeater** — manually testing URLs, headers, and cache behavior.
- **Burp Intruder** — fuzzing delimiters and special characters.
- **Burp Comparer** — comparing responses.
- **Browser** — validating application behavior.
- **Exploit Server** — delivering payloads to the victim.
- **HTTP headers** — analyzing `X-Cache`, `Cache-Control`, `Age`, and `Vary`.

---

## 📝 Walkthrough Format

Each report explains:

```text
What are we testing?
Why does it matter?
How does the origin behave?
How does the cache behave?
Where does the discrepancy appear?
How do we build the exploit chain?
How can this be prevented?
```

Reports usually include:

- lab goal;
- key idea;
- step-by-step walkthrough;
- origin/cache behavior analysis;
- core discrepancy;
- mitigation recommendations;
- checklist;
- hidden solution using `<details>`.

---

## 📂 Directory Structure

```text
web_cache_deception/
│
├── README.md
├── README_EN.md
│
├── 001_ru_exploiting_path_mapping_for_web_cache_deception.md
├── 001_en_exploiting_path_mapping_for_web_cache_deception.md
│
├── 002_ru_exploiting_path_delimiters_for_web_cache_deception.md
├── 002_en_exploiting_path_delimiters_for_web_cache_deception.md
│
├── 003_ru_exploiting_origin_server_normalization_for_web_cache_deception.md
├── 003_en_exploiting_origin_server_normalization_for_web_cache_deception.md
│
├── 004_ru_exploiting_cache_server_normalization_for_web_cache_deception.md
└── 004_en_exploiting_cache_server_normalization_for_web_cache_deception.md
```

---

## 🎓 What You Will Learn

After completing this section, you will be able to:

- understand how shared caching works;
- analyze `X-Cache: miss` and `X-Cache: hit`;
- identify sensitive endpoints;
- test path mapping behavior;
- discover path delimiters;
- analyze URL normalization;
- find parser discrepancies between the origin and the cache;
- build Web Cache Deception exploit chains;
- use cache busters correctly;
- explain why private responses must not be cached;
- write practical mitigation recommendations for CDNs and reverse proxies.

---

## 🛡️ Web Cache Deception Prevention

Primary defense:

```http
Cache-Control: no-store, private
```

Also important:

- do not cache authenticated endpoints;
- use `Vary: Cookie` when responses depend on cookies;
- do not make caching decisions based only on `.js`, `.css`, or `.png` extensions;
- avoid broad prefix-based cache rules without strict validation;
- keep URL normalization consistent between the cache and the origin;
- enforce strict routing;
- return `404` or `403` for unexpected suffixes;
- canonicalize URLs before cache lookup;
- test encoded delimiters such as `%2f`, `%23`, `%3f`, and `%2e`;
- test cache behavior in staging before production.

---

## 🔗 Useful Links

- PortSwigger Web Cache Deception: https://portswigger.net/web-security/web-cache-deception
- Web Cache Deception labs: https://portswigger.net/web-security/all-labs#web-cache-deception
- Burp Suite: https://portswigger.net/burp

---

## 🚀 Next Steps

After Web Cache Deception, logical next sections are:

- Web Cache Poisoning;
- HTTP Request Smuggling;
- Access Control;
- SSRF;
- XSS.

---

⬆ [Back to top](#-web-cache-deception--portswigger-web-security-academy)
