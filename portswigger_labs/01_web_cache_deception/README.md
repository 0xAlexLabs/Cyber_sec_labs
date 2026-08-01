# 🧊 Web Cache Deception — PortSwigger Web Security Academy

<p align="center">
  <img src="https://img.shields.io/badge/Platform-PortSwigger-red" />
  <img src="https://img.shields.io/badge/Topic-Web%20Cache%20Deception-blue" />
  <img src="https://img.shields.io/badge/Labs-4-success" />
  <img src="https://img.shields.io/badge/Progress-100%25-brightgreen" />
  <img src="https://img.shields.io/badge/Language-RU%20%7C%20EN-purple" />
</p>

<p align="right">
🇬🇧 <a href="./README_EN.md"><b>English Version</b></a>
</p>

> 📂 Путь: `cyber_sec_labs/portswigger_labs/web_cache_deception`  
> 📚 Раздел: Web Cache Deception  
> 🧪 Лабораторий: 4  
> 📝 Отчетов: 8 — русская и английская версии для каждой лабораторной

---

## 📚 Содержание

- [О разделе](#-о-разделе)
- [Важное предупреждение](#️-важное-предупреждение)
- [Прогресс](#-прогресс)
- [Roadmap обучения](#️-roadmap-обучения)
- [Карта лабораторных](#-карта-лабораторных)
- [Что такое Web Cache Deception](#-что-такое-web-cache-deception)
- [Ключевые понятия](#-ключевые-понятия)
- [Основные техники](#-основные-техники)
- [Инструменты](#-инструменты)
- [Формат walkthrough](#-формат-walkthrough)
- [Структура каталога](#-структура-каталога)
- [Что дает прохождение раздела](#-что-дает-прохождение-раздела)
- [Защита от Web Cache Deception](#️-защита-от-web-cache-deception)
- [Полезные ссылки](#-полезные-ссылки)
- [Что дальше](#-что-дальше)

---

## 📌 О разделе

Этот каталог содержит разборы лабораторных работ **PortSwigger Web Security Academy** по теме **Web Cache Deception**.

Цель раздела — понять, как из-за расхождений между **origin server** и **cache server** персональные данные пользователя могут попасть в shared cache и стать доступными атакующему.

В этих лабораторных основная идея не в том, чтобы "сломать" backend напрямую, а в том, чтобы заставить разные слои инфраструктуры по-разному понять один и тот же URL.

---

## ⚠️ Важное предупреждение

Материалы предназначены **только для обучения**.

Все действия выполнялись исключительно в легальной учебной среде PortSwigger Web Security Academy. Не используйте эти техники против реальных систем без явного письменного разрешения владельца.

---

## 📊 Прогресс

```text
Раздел: Web Cache Deception
Лабораторий: 4 / 4
Отчетов: 8
Языки: RU / EN
Статус: завершено
```

```text
████ 4 / 4 — 100%
```

---

## 🗺️ Roadmap обучения

```text
Web Cache Deception
│
├── Path Mapping
│   └── origin игнорирует хвост пути, cache видит статический ресурс
│
├── Path Delimiters
│   └── origin и cache по-разному понимают разделители URL
│
├── Origin Server Normalization
│   └── origin нормализует путь иначе, чем cache
│
├── Cache Server Normalization
│   └── cache нормализует путь иначе, чем origin
│
└── Defense
    ├── no-store / private
    ├── Vary: Cookie
    ├── strict routing
    ├── consistent normalization
    └── safe cache rules
```

---

## 📋 Карта лабораторных

| № | Лабораторная | Чему учит | RU | EN |
|---|--------------|-----------|:--:|:--:|
| 001 | **Эксплуатация path mapping для Web Cache Deception**<br><sub>Exploiting Path Mapping for Web Cache Deception</sub> | расхождение между dynamic endpoint на origin и static-looking URL в cache | [RU](./001_ru_exploiting_path_mapping_for_web_cache_deception.md) | [EN](./001_en_exploiting_path_mapping_for_web_cache_deception.md) |
| 002 | **Эксплуатация path delimiters для Web Cache Deception**<br><sub>Exploiting Path Delimiters for Web Cache Deception</sub> | поиск delimiter'ов, которые origin и cache интерпретируют по-разному | [RU](./002_ru_exploiting_path_delimiters_for_web_cache_deception.md) | [EN](./002_en_exploiting_path_delimiters_for_web_cache_deception.md) |
| 003 | **Эксплуатация нормализации на origin server**<br><sub>Exploiting Origin Server Normalization for Web Cache Deception</sub> | различия в URL normalization: origin нормализует путь, cache применяет правило кеширования иначе | [RU](./003_ru_exploiting_origin_server_normalization_for_web_cache_deception.md) | [EN](./003_en_exploiting_origin_server_normalization_for_web_cache_deception.md) |
| 004 | **Эксплуатация нормализации на cache server**<br><sub>Exploiting Cache Server Normalization for Web Cache Deception</sub> | parser discrepancy: cache нормализует путь иначе, чем origin, и кеширует приватный ответ | [RU](./004_ru_exploiting_cache_server_normalization_for_web_cache_deception.md) | [EN](./004_en_exploiting_cache_server_normalization_for_web_cache_deception.md) |

---

## 🧠 Что такое Web Cache Deception

**Web Cache Deception (WCD)** — это уязвимость, при которой атакующий заставляет cache сохранить приватный ответ пользователя как будто это публичный статический ресурс.

Упрощенная схема:

```text
Victim открывает специально подготовленный URL
        ↓
Origin server возвращает приватную страницу
        ↓
Cache считает URL статическим ресурсом
        ↓
Cache сохраняет приватный response
        ↓
Attacker открывает тот же URL и получает данные victim
```

Главная проблема:

```text
Origin server и cache server по-разному интерпретируют URL.
```

---

## 🧩 Ключевые понятия

### Origin server

Это основной backend-сервер приложения. Именно он знает бизнес-логику, сессии, cookies и приватные endpoints вроде:

```text
/my-account
/profile
/settings
/api/me
```

### Cache server

Это промежуточный слой: CDN, reverse proxy или caching proxy. Он решает, можно ли сохранить response и отдать его повторно без обращения к origin.

### Cache key

Это ключ, по которому cache хранит response. Обычно он строится из URL и иногда учитывает headers, query string или cookies.

### Static-looking URL

URL, который выглядит как статический ресурс:

```text
/file.js
/image.png
/styles.css
/resources/...
```

Если cache слишком доверяет расширению или префиксу, он может ошибочно сохранить приватный response.

### Parser discrepancy

Это расхождение в парсинге URL:

```text
Origin видит одно
Cache видит другое
```

Именно parser discrepancy лежит в основе всех лабораторных этого раздела.

### Cache buster

Уникальная строка в URL, которая создает новый cache key:

```text
?wcd123
/random.js
```

Нужна, чтобы victim первым заполнил cache своими данными, а не attacker.

---

## 🧪 Основные техники

### 1. Path Mapping

Origin воспринимает:

```text
/my-account/abc.js
```

как:

```text
/my-account
```

А cache видит `.js` и считает response статикой.

### 2. Path Delimiters

Некоторые символы могут быть delimiter'ами для origin, но не для cache.

Пример:

```text
/my-account;test.js
```

Origin может обработать это как `/my-account`, а cache — как `.js` resource.

### 3. Origin Server Normalization

Origin декодирует и нормализует путь:

```text
/resources/..%2fmy-account
→ /my-account
```

А cache продолжает применять правило кеширования по `/resources`.

### 4. Cache Server Normalization

Cache нормализует путь и видит `/resources`, а origin из-за delimiter'а продолжает отдавать `/my-account`.

Пример идеи:

```text
Origin → /my-account
Cache  → /resources
```

### 5. Cache Poisoning / Cache Deception Chain

В WCD важно, чтобы victim сделал первый запрос к новому cache key. Тогда в cache попадет именно приватный response victim.

---

## 🛠 Инструменты

- **Burp Proxy** — перехват и анализ запросов.
- **Burp Repeater** — ручная проверка URL, headers и cache behavior.
- **Burp Intruder** — fuzzing delimiter'ов и специальных символов.
- **Burp Comparer** — сравнение ответов.
- **Browser** — проверка поведения приложения.
- **Exploit Server** — доставка payload victim.
- **HTTP headers** — анализ `X-Cache`, `Cache-Control`, `Age`, `Vary`.

---

## 📝 Формат walkthrough

Каждый отчет построен так, чтобы объяснить:

```text
Что проверяем?
Почему это важно?
Как ведет себя origin?
Как ведет себя cache?
Где появляется discrepancy?
Как построить exploit chain?
Как защититься?
```

Обычно внутри отчета есть:

- цель лаборатории;
- ключевая идея;
- пошаговое прохождение;
- анализ origin/cache behavior;
- главный discrepancy;
- рекомендации по защите;
- чеклист;
- скрытое решение через `<details>`.

---

## 📂 Структура каталога

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

## 🎓 Что дает прохождение раздела

После прохождения раздела вы сможете:

- понимать, как работает shared cache;
- анализировать `X-Cache: miss` и `X-Cache: hit`;
- находить чувствительные endpoints;
- проверять path mapping;
- искать path delimiters;
- анализировать URL normalization;
- находить parser discrepancies между origin и cache;
- строить Web Cache Deception exploit chain;
- использовать cache buster;
- объяснить, почему нельзя кешировать приватные responses;
- формулировать рекомендации по защите CDN и reverse proxy.

---

## 🛡️ Защита от Web Cache Deception

Главная защита:

```http
Cache-Control: no-store, private
```

Также важно:

- не кешировать authenticated endpoints;
- использовать `Vary: Cookie`, если response зависит от cookies;
- не принимать решение о кешировании только по расширению `.js`, `.css`, `.png`;
- не использовать грубые prefix-based cache rules без строгой проверки;
- синхронизировать URL normalization между cache и origin;
- строго валидировать маршруты;
- возвращать `404` или `403` для неожиданных suffixes;
- canonicalize URL до cache lookup;
- проверять encoded delimiters: `%2f`, `%23`, `%3f`, `%2e`;
- тестировать cache behavior в staging перед production.

---

## 🔗 Полезные ссылки

- PortSwigger Web Cache Deception: https://portswigger.net/web-security/web-cache-deception
- Web Cache Deception labs: https://portswigger.net/web-security/all-labs#web-cache-deception
- Burp Suite: https://portswigger.net/burp

---

## 🚀 Что дальше

После Web Cache Deception логично продолжать:

- Web Cache Poisoning;
- HTTP Request Smuggling;
- Access Control;
- SSRF;
- XSS.

---

⬆ [Наверх](#-web-cache-deception--portswigger-web-security-academy)
