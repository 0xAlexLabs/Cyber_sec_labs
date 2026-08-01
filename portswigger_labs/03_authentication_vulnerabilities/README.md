# 🔐 Authentication — PortSwigger Web Security Academy

<p align="center">
  <img src="https://img.shields.io/badge/Platform-PortSwigger-red" />
  <img src="https://img.shields.io/badge/Topic-Authentication-blue" />
  <img src="https://img.shields.io/badge/Labs-11-success" />
  <img src="https://img.shields.io/badge/Progress-100%25-brightgreen" />
  <img src="https://img.shields.io/badge/Language-RU%20%7C%20EN-purple" />
</p>

<p align="right">
🇬🇧 <a href="./README_EN.md"><b>English Version</b></a>
</p>

> 📂 Путь: `cyber_sec_labs/portswigger_labs/authentication`  
> 📚 Раздел: Authentication vulnerabilities  
> 🧪 Лабораторий: 11  
> 📝 Отчетов: 22 — русская и английская версии для каждой лабораторной

---

## 📚 Содержание

- [О разделе](#-о-разделе)
- [Важное предупреждение](#️-важное-предупреждение)
- [Прогресс](#-прогресс)
- [Roadmap обучения](#️-roadmap-обучения)
- [Карта лабораторных](#-карта-лабораторных)
- [Что изучается](#-что-изучается)
- [Техники атак на аутентификацию](#-техники-атак-на-аутентификацию)
- [Поверхности атаки](#-поверхности-атаки)
- [Инструменты](#-инструменты)
- [Формат walkthrough](#-формат-walkthrough)
- [Структура каталога](#-структура-каталога)
- [Что дает прохождение раздела](#-что-дает-прохождение-раздела)
- [Защита механизмов аутентификации](#️-защита-механизмов-аутентификации)
- [Полезные ссылки](#-полезные-ссылки)
- [Что дальше](#-что-дальше)

---

## 📌 О разделе

Этот каталог содержит подробные разборы лабораторных работ **PortSwigger Web Security Academy** по теме **Authentication vulnerabilities**.

Цель раздела — не просто собрать финальные решения, а сформировать практическую методологию проверки аутентификации:

- находить Username Enumeration по тексту, длине, статусу, времени ответа и блокировке;
- анализировать brute-force protection, rate limiting и account lockout;
- проверять, действительно ли сервер требует прохождение второго фактора;
- исследовать связь между первым фактором, MFA-кодом, сессией и пользователем;
- reverse engineer persistent authentication cookies;
- отличать Base64-кодирование от криптографической защиты;
- анализировать password reset flow, reset tokens и доверие к proxy headers;
- объяснять уязвимость на уровне backend-логики;
- формулировать практические рекомендации по защите.

Каждый walkthrough предназначен для повторения, подготовки к собеседованиям и формирования собственной базы знаний по web pentest.

---

## ⚠️ Важное предупреждение

Материалы предназначены **только для обучения**.

Все действия выполнялись исключительно в легальной учебной среде PortSwigger Web Security Academy. Не используйте brute force, перехват токенов, подмену cookies или password reset атаки против реальных систем без явного письменного разрешения владельца.

---

## 📊 Прогресс

```text
Раздел: Authentication vulnerabilities
Лабораторий: 11 / 11
Отчетов: 22
Языки: RU / EN
Статус: завершено
```

```text
███████████ 11 / 11 — 100%
```

---

## 🗺️ Roadmap обучения

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

## 📋 Карта лабораторных

| № | Лабораторная | Чему учит | RU | EN |
|---|--------------|-----------|:--:|:--:|
| 001 | **Перечисление пользователей по различающимся ответам**<br><sub>Username enumeration via different responses</sub> | разные ошибки входа, двухэтапный brute force, анализ Status и Length | [RU](./001_ru_username_enumeration_via_different_responses.md) | [EN](./001_en_username_enumeration_via_different_responses.md) |
| 002 | **Перечисление пользователей по едва заметным различиям**<br><sub>Username enumeration via subtly different responses</sub> | Grep - Extract, поиск различия в точке и пробеле, сравнение почти одинаковых ответов | [RU](./002_ru_username_enumeration_via_subtly_different_responses.md) | [EN](./002_en_username_enumeration_via_subtly_different_responses.md) |
| 003 | **Перечисление пользователей по времени ответа**<br><sub>Username enumeration via response timing</sub> | timing side channel, длинный пароль, обход IP-блокировки через `X-Forwarded-For` | [RU](./003_ru_username_enumeration_via_response_timing.md) | [EN](./003_en_username_enumeration_via_response_timing.md) |
| 004 | **Обход защиты от brute force через IP block**<br><sub>Broken brute-force protection, IP block</sub> | сброс счётчика успешным входом, Pitchfork, Resource Pool = 1 | [RU](./004_ru_broken_bruteforce_protection_ip_block.md) | [EN](./004_en_broken_bruteforce_protection_ip_block.md) |
| 005 | **Перечисление пользователей через блокировку аккаунта**<br><sub>Username enumeration via account lock</sub> | account-lock side channel, Cluster Bomb, Null Payloads | [RU](./005_ru_username_enumeration_via_account_lock.md) | [EN](./005_en_username_enumeration_via_account_lock.md) |
| 006 | **Простой обход двухфакторной аутентификации**<br><sub>2FA simple bypass</sub> | пропуск `/login2`, прямой переход к `/my-account`, отсутствие server-side enforcement | [RU](./006_ru_2fa_simple_bypass.md) | [EN](./006_en_2fa_simple_bypass.md) |
| 007 | **Нарушенная логика двухфакторной аутентификации**<br><sub>2FA broken logic</sub> | подмена `verify`, генерация кода для жертвы, brute force MFA-кода | [RU](./007_ru_2fa_broken_logic.md) | [EN](./007_en_2fa_broken_logic.md) |
| 008 | **Подбор cookie постоянного входа**<br><sub>Brute-forcing a stay-logged-in cookie</sub> | Base64, MD5, Payload Processing, подделка persistent cookie | [RU](./008_ru_brute_forcing_stay_logged_in_cookie.md) | [EN](./008_en_brute_forcing_stay_logged_in_cookie.md) |
| 009 | **Офлайн-восстановление пароля**<br><sub>Offline password cracking</sub> | Stored XSS, кража cookie, извлечение MD5 и offline cracking | [RU](./009_ru_offline_password_cracking.md) | [EN](./009_en_offline_password_cracking.md) |
| 010 | **Нарушенная логика сброса пароля**<br><sub>Password reset broken logic</sub> | подмена hidden `username`, пустой token, смена пароля жертвы | [RU](./010_ru_password_reset_broken_logic.md) | [EN](./010_en_password_reset_broken_logic.md) |
| 011 | **Отравление ссылки сброса пароля через middleware**<br><sub>Password reset poisoning via middleware</sub> | `X-Forwarded-Host`, абсолютная reset-ссылка, перехват токена | [RU](./011_ru_password_reset_poisoning_via_middleware.md) | [EN](./011_en_password_reset_poisoning_via_middleware.md) |

---

## 🧠 Что изучается

### 1. Password-Based Authentication

Базовая форма выглядит просто:

```text
username + password
```

Но для пентестера важен не только текст сообщения об ошибке. Анализируется весь наблюдаемый результат:

```text
HTTP status
Response body
Response length
Response time
Redirect target
Set-Cookie
Account lock behavior
```

Главный принцип:

```text
Сравнивать нужно не только сообщение,
а полный HTTP-ответ и состояние приложения.
```

---

### 2. Username Enumeration

Username Enumeration подтверждает существование аккаунта до подбора пароля.

В разделе рассматриваются четыре канала:

```text
1. Явно разные сообщения.
2. Почти одинаковые сообщения с одним отличием.
3. Различие времени ответа.
4. Блокировка только существующего аккаунта.
```

Это резко уменьшает пространство поиска:

```text
100 usernames × 100 passwords = 10 000 комбинаций
100 usernames + 100 passwords = 200 запросов
```

После нахождения username он фиксируется, а Intruder перебирает только пароли.

---

### 3. Subtle Response Differences

Различие может состоять всего в одном символе:

```text
Invalid username or password.
Invalid username or password 
```

Глазами его легко пропустить. Поэтому используются:

- Grep - Match;
- Grep - Extract;
- сортировка по Length;
- сравнение Status;
- выделение фрагмента HTML;
- поиск отклонений от основной группы ответов.

Один пробел, точка, тег или дополнительный блок HTML могут выступать полноценным side channel.

---

### 4. Timing Side Channels

Даже при одинаковом теле ответа сервер может обрабатывать запросы по-разному:

```text
Несуществующий username
        ↓
Быстрый отказ

Существующий username
        ↓
Вычисление password hash
        ↓
Более медленный ответ
```

Надёжный timing-анализ требует:

- нескольких повторных измерений;
- контроля сетевого шума;
- ограниченного параллелизма;
- длинного password для усиления различия;
- изменения `X-Forwarded-For`, если блокировка привязана к IP;
- поиска устойчивой задержки, а не одного случайного выброса.

---

### 5. Brute-Force Protection

Защита от brute force может сама содержать логическую ошибку.

Уязвимый сценарий:

```text
Неверная попытка увеличивает счётчик
        ↓
Успешный вход в собственный аккаунт сбрасывает счётчик
        ↓
Атакующий чередует запросы
        ↓
Блокировка не наступает
```

Другой сценарий:

```text
Только существующий аккаунт блокируется
        ↓
Сообщение о блокировке подтверждает username
```

Rate limiting не должен зависеть от данных, которые атакующий полностью контролирует, и не должен раскрывать существование учётной записи.

---

### 6. Burp Intruder Strategy

В лабораториях используются разные attack types.

**Sniper** подходит, когда изменяется одна позиция:

```text
password=§candidate§
mfa-code=§0000§
```

**Pitchfork** синхронно берёт значения из нескольких списков по одинаковому индексу. Это полезно, когда нужно чередовать:

```text
wiener:peter
carlos:candidate1
wiener:peter
carlos:candidate2
```

**Cluster Bomb** перебирает декартово произведение наборов. В сочетании с Null Payloads он позволяет повторить каждый username несколько раз и вызвать account lock.

---

### 7. Multi-Factor Authentication

Страница MFA ещё не означает, что MFA реально enforced.

Уязвимый поток:

```text
Пароль принят
        ↓
Создана полноценная сессия
        ↓
Показана форма /login2
        ↓
/my-account уже доступен
```

Безопасный поток:

```text
Пароль принят
        ↓
Создана ограниченная pre-auth session
        ↓
MFA-код принят
        ↓
Сессия повышена до authenticated
        ↓
Открыты защищённые endpoints
```

Проверка завершения MFA должна выполняться сервером для каждого защищённого запроса.

---

### 8. Broken 2FA Logic

В уязвимой реализации клиент выбирает аккаунт для проверки:

```http
Cookie: verify=wiener
```

После подмены:

```http
Cookie: verify=carlos
```

сервер генерирует и проверяет код для Carlos, хотя пароль Carlos не вводился.

Безопасный инвариант:

```text
Identity второго фактора должна определяться только
из server-side pre-auth session первого фактора.
```

Нельзя доверять username из cookie, hidden input, URL или POST body.

---

### 9. MFA Code Brute Force

Четырёхзначный код имеет ограниченное пространство:

```text
0000–9999
```

Для перебора в Intruder:

```text
Payload type: Numbers
From: 0
To: 9999
Min integer digits: 4
```

Защита должна включать:

- короткий срок жизни;
- лимит попыток;
- блокировку или progressive delay;
- одноразовость;
- инвалидирование старого кода после выпуска нового;
- привязку к конкретной pre-auth session.

---

### 10. Persistent Authentication Cookies

`Stay logged in` создаёт долгоживущий механизм аутентификации.

Уязвимая формула:

```text
Base64(username:MD5(password))
```

Проблемы:

- Base64 обратим;
- username известен;
- MD5 быстр;
- алгоритм детерминирован;
- cookie можно построить локально для каждого password candidate;
- password hash фактически передаётся клиенту.

Безопасная cookie должна содержать случайный opaque token, который можно отозвать и который не связан напрямую с паролем.

---

### 11. Online и Offline Cracking

Online brute force:

```text
candidate → HTTP request → response
```

Ограничивается rate limiting, сетью и журналированием.

Offline cracking:

```text
stolen hash → local dictionary → plaintext
```

После получения хэша сервер больше не участвует в каждой попытке.

Цепочка лаборатории:

```text
Stored XSS
        ↓
Кража stay-logged-in cookie
        ↓
Base64 decode
        ↓
username:MD5(password)
        ↓
Offline cracking
        ↓
Вход как жертва
```

---

### 12. Password Reset Logic

Password reset — альтернативный механизм аутентификации.

Уязвимая логика:

```text
Token проверяется отдельно
Username берётся из hidden-поля
Сервер меняет пароль переданного username
```

Подмена:

```http
username=wiener
```

на:

```http
username=carlos
```

Безопасная схема:

```text
reset_token → server-side record → user_id → reset password
```

Username из клиента не должен определять целевой аккаунт.

---

### 13. Password Reset Poisoning

Middleware может доверять прокси-заголовку:

```http
X-Forwarded-Host: attacker.example
```

и создать ссылку:

```text
https://attacker.example/forgot-password?temp-forgot-password-token=...
```

После перехода жертвы token попадает на exploit server.

Полная цепочка:

```text
Reset request с X-Forwarded-Host
        ↓
Письмо с доменом атакующего
        ↓
Переход жертвы
        ↓
Токен в access log
        ↓
Смена пароля жертвы
```

---

## 🧩 Техники атак на аутентификацию

- **Username Enumeration by Response** — поиск username по тексту, статусу, длине или HTML.
- **Grep - Extract Analysis** — автоматическое выделение едва заметного отличия.
- **Timing Enumeration** — поиск существующего пользователя по задержке.
- **IP Lockout Bypass** — обход IP-based защиты через доверенные proxy headers.
- **Counter Reset Abuse** — сброс brute-force счётчика успешным входом.
- **Account Lock Oracle** — использование блокировки как признака существования аккаунта.
- **Password Brute Force** — перебор паролей для найденного username.
- **MFA Step Skipping** — прямой доступ к защищённой странице до завершения MFA.
- **MFA Identity Confusion** — подмена пользователя, для которого проверяется код.
- **MFA Code Brute Force** — перебор небольшого числового пространства.
- **Persistent Cookie Forgery** — воспроизведение слабого алгоритма remember-me cookie.
- **Offline Hash Cracking** — локальное восстановление пароля по украденному hash.
- **Password Reset Parameter Tampering** — изменение target username.
- **Host Header Injection** — влияние на абсолютные URL через Host-related headers.
- **Password Reset Poisoning** — направление reset token на атакующий домен.
- **Session State Analysis** — сравнение anonymous, pre-auth, MFA-pending и authenticated states.

---

## 🧭 Поверхности атаки

| Поверхность | Что проверяется |
|-------------|-----------------|
| `POST /login` | сообщения, Status, Length, timing, redirects, cookies |
| `username` | enumeration, регистр, пробелы, Unicode, нормализация |
| `password` | brute force, длина, truncation, timing, policy |
| `X-Forwarded-For` | обход IP rate limiting и доверие к proxy headers |
| `POST /login2` | привязка к первому фактору, лимит попыток, reuse |
| `verify` | возможность выбрать чужой аккаунт для MFA |
| `session` | момент полной аутентификации, rotation, logout invalidation |
| `stay-logged-in` | структура, энтропия, подпись, срок жизни, отзыв |
| `forgot-password` | enumeration, token generation, expiry, one-time use |
| hidden `username` | доверие к изменяемому client-side полю |
| `Host` / `X-Forwarded-Host` | построение абсолютных ссылок |
| reset token | энтропия, user binding, expiry, replay |

---

## 🛠 Инструменты

- **Burp Proxy** — перехват login, MFA, cookie и password reset запросов.
- **Burp Repeater** — ручная проверка логики, headers, cookies и parameters.
- **Burp Intruder** — enumeration, password brute force и перебор MFA-кодов.
- **Grep - Match / Grep - Extract** — поиск различий в ответах.
- **Payload Processing** — MD5, prefix/suffix, Base64 и URL encoding.
- **Resource Pool** — контроль параллелизма и порядка запросов.
- **Burp Decoder** — Base64 и анализ persistent cookies.
- **Exploit Server** — получение украденных токенов и XSS callbacks.
- **Email Client** — исследование reset links.
- **Browser Developer Tools** — cookies, redirects и storage.
- **Python / shell utilities** — локальная проверка форматов и hashes в учебной среде.

---

## 📝 Формат walkthrough

Каждый разбор объясняет не только **что отправить**, но и **почему сервер принимает запрос**.

Обычно внутри отчета есть:

- цель лаборатории;
- краткая теория;
- корректный и уязвимый поток;
- пошаговое прохождение;
- реальные HTTP-запросы;
- настройка Burp Intruder;
- использованные payload'ы;
- анализ Status, Length, timing и redirects;
- разбор backend-логики;
- раздел «Как думать как пентестер»;
- типичные ошибки;
- рекомендации по защите;
- итоговый чек-лист;
- скрытые credentials через `<details>`.

---

## 📂 Структура каталога

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

## 🎓 Что дает прохождение раздела

После прохождения раздела вы сможете:

- системно тестировать login flow;
- находить Username Enumeration через текст, длину, статус, timing и account lock;
- выбирать Sniper, Pitchfork или Cluster Bomb под задачу;
- использовать Grep - Match и Grep - Extract;
- анализировать rate limiting и account lockout;
- выявлять обходы IP-блокировки и сброс счётчика;
- проверять server-side enforcement второго фактора;
- находить MFA identity confusion;
- настраивать перебор кодов с ведущими нулями;
- reverse engineer persistent authentication cookies;
- отличать Base64 от encryption и signing;
- объяснять риски MD5 и offline cracking;
- анализировать reset tokens и user binding;
- тестировать Host Header Injection и reset poisoning;
- объяснять уязвимость на уровне backend state machine;
- предлагать практические меры защиты.

---

## 🛡️ Защита механизмов аутентификации

### Единообразные ответы

Для несуществующего пользователя и неверного пароля должны совпадать:

```text
HTTP status
Response body
Response length
Response time
Redirect behavior
```

Одинакового текста недостаточно, если отличается timing или структура HTML.

### Rate Limiting и Account Lockout

- лимиты по аккаунту и IP;
- progressive delays;
- дополнительные device/session signals;
- мониторинг credential stuffing;
- безопасная процедура разблокировки;
- уведомления владельца;
- доверие к proxy headers только от известного reverse proxy;
- отсутствие логических reset-путей счётчика.

### Безопасная MFA

- ограниченная pre-auth session после пароля;
- target user ID только на сервере;
- запрет выбора identity через cookie или parameter;
- проверка MFA-state на каждом защищённом endpoint;
- короткий срок жизни и одноразовость кода;
- лимит попыток;
- инвалидирование предыдущего кода;
- session rotation после MFA.

### Persistent Login

- случайные opaque tokens;
- хранение server-side hash токена;
- отсутствие password hash в cookie;
- expiration и per-device revocation;
- rotation после использования;
- отзыв после смены пароля;
- `Secure`, `HttpOnly`, подходящий `SameSite`.

### Password Storage

Использовать:

```text
Argon2id
scrypt
bcrypt
PBKDF2
```

Не использовать:

```text
MD5
SHA-1
обычный SHA-256 без salt и work factor
```

### Password Reset

- high-entropy random token;
- server-side hash token;
- строгая связь `token → user_id`;
- одноразовость и короткий expiry;
- invalidation старых tokens;
- trusted base URL из конфигурации;
- запрет доверия к внешним `Host` и `X-Forwarded-Host`;
- отзыв sessions и persistent tokens после сброса;
- одинаковый ответ для существующего и несуществующего аккаунта.

---

## 🔗 Полезные ссылки

- PortSwigger Authentication: https://portswigger.net/web-security/authentication
- Password-Based Authentication: https://portswigger.net/web-security/authentication/password-based
- Multi-Factor Authentication: https://portswigger.net/web-security/authentication/multi-factor
- Other Authentication Mechanisms: https://portswigger.net/web-security/authentication/other-mechanisms
- Burp Intruder: https://portswigger.net/burp/documentation/desktop/tools/intruder
- Web Security Academy: https://portswigger.net/web-security

---

## 🚀 Что дальше

После Authentication логично продолжать:

- Access Control;
- OAuth Authentication;
- JWT;
- Business Logic Vulnerabilities;
- Cross-Site Scripting;
- CSRF;
- Web Cache Poisoning;
- Server-Side Vulnerabilities.

---

⬆ [Наверх](#-authentication--portswigger-web-security-academy)
