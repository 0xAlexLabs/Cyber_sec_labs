# 🛡️ PortSwigger Web Cache Deception Labs

PortSwigger Web Security Academy — это обучающая платформа по веб-безопасности, где можно практиковаться в поиске и эксплуатации реальных веб-уязвимостей.

В этом репозитории собраны подробные walkthrough по лабораторным работам из раздела:

```text
Web Cache Deception
```

Цель этих заметок:
- закрепить понимание работы cache;
- научиться находить parser discrepancies;
- разобраться в cache poisoning и cache deception;
- понять различия между cache и origin server;
- тренировать практические навыки эксплуатации веб-уязвимостей.

---

# ⚠️ Важно

Все лабораторные работы выполняются исключительно в рамках платформы PortSwigger Web Security Academy.

В репозитории:
- не публикуются реальные уязвимости;
- не используются реальные цели;
- весь материал предназначен только для обучения.

---

# 🔗 Платформа

https://portswigger.net/web-security

---

# 📚 Лабораторные работы

## 🧩 Web Cache Deception

### 001 — Exploiting path mapping for web cache deception

- [RU](./001_ru_exploiting_path_mapping_for_web_cache_deception.md)
- [EN](./001_en_exploiting_path_mapping_for_web_cache_deception.md)

---

### 002 — Exploiting path delimiters for web cache deception

- [RU](./002_ru_exploiting_path_delimiters_for_web_cache_deception.md)
- [EN](./002_en_exploiting_path_delimiters_for_web_cache_deception.md)

---

### 003 — Exploiting origin server normalization for web cache deception

- [RU](./003_ru_exploiting_origin_server_normalization_for_web_cache_deception.md)
- [EN](./003_en_exploiting_origin_server_normalization_for_web_cache_deception.md)

---

### 004 — Exploiting cache server normalization for web cache deception

- [RU](./004_ru_exploiting_cache_server_normalization_for_web_cache_deception.md)
- [EN](./004_en_exploiting_cache_server_normalization_for_web_cache_deception.md)

---

# 🧠 Что изучается в этих лабах

В процессе прохождения лабораторных работ разбираются:

- cache poisoning;
- web cache deception;
- path normalization;
- parser discrepancies;
- delimiter discrepancies;
- URL canonicalization;
- cache rules;
- origin vs cache behavior;
- path traversal normalization;
- CDN/cache logic.

---

# 🛠 Используемые инструменты

Во время прохождения лабораторных работ используются:

- Burp Suite
- Repeater
- Intruder
- Proxy
- Decoder
- браузер Burp Suite

---

# 📖 Формат walkthrough

Каждая лабораторная работа содержит:

- пошаговое объяснение;
- логику поиска уязвимости;
- разбор запросов;
- объяснение механики атаки;
- payload examples;
- рекомендации по защите;
- чеклист исследователя.

---

# 🌍 Языки

Все walkthrough доступны:

- 🇷🇺 на русском языке
- 🇬🇧 на английском языке

Это помогает:
- лучше закреплять материал;
- тренировать технический английский;
- структурировать знания по web security.

---

# 🚀 Цель репозитория

Главная цель — не просто получить solution,
а научиться:

- мыслить как исследователь;
- понимать поведение веб-инфраструктуры;
- находить различия в обработке запросов;
- строить exploit chain самостоятельно.
