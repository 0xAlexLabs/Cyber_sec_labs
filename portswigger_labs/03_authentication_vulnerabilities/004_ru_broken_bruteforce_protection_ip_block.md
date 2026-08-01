# 📘 PortSwigger Lab: Broken Brute-force Protection, IP Block

<a id="top"></a>

> 🔗 **Лабораторная работа:** https://portswigger.net/web-security/authentication/password-based/lab-broken-bruteforce-protection-ip-block  
> 📄 **Candidate passwords:** https://portswigger.net/web-security/authentication/auth-lab-passwords  
> 🎯 **Тема:** Authentication vulnerabilities — Flawed Brute-force Protection / IP Block Logic Flaw  
> 🧪 **Уровень:** Practitioner  
> 👤 **Наш аккаунт:** `wiener:peter`  
> 🎯 **Целевой пользователь:** `carlos`

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🔐 Password-based Authentication](#password-auth)
- [💣 Brute-force, Credential Stuffing и Password Spraying](#bruteforce-types)
- [🛡 Как обычно защищаются от brute-force](#protection)
- [🧩 Ключевая идея лаборатории](#idea)
- [💥 В чем логическая ошибка](#logic-flaw)
- [🧪 Почему эта лаборатория важна](#importance)
- [🎯 Почему используется Pitchfork](#pitchfork)
- [🚫 Почему не Sniper](#not-sniper)
- [🚫 Почему не Cluster Bomb](#not-cluster-bomb)
- [⚙ Почему нужен Resource Pool = 1](#resource-pool)
- [🔍 Шаг 1 — Изучаем страницу логина](#step1)
- [🔍 Шаг 2 — Проверяем блокировку после ошибок](#step2)
- [🔍 Шаг 3 — Проверяем сброс счетчика через свой аккаунт](#step3)
- [🔍 Шаг 4 — Отправляем запрос в Intruder](#step4)
- [🔍 Шаг 5 — Настраиваем Pitchfork](#step5)
- [🔍 Шаг 6 — Готовим username payload list](#step6)
- [🔍 Шаг 7 — Готовим password payload list](#step7)
- [🔍 Шаг 8 — Настраиваем Resource Pool](#step8)
- [🔍 Шаг 9 — Запускаем атаку](#step9)
- [🔍 Шаг 10 — Находим пароль Carlos](#step10)
- [🔍 Шаг 11 — Входим в аккаунт](#step11)
- [🧩 Использованные Payloads](#payloads)
- [🔬 Почему атака сработала](#breakdown)
- [🌍 Практическое применение](#real-world)
- [🧠 Как думать как пентестер](#pentester)
- [❌ Типичные ошибки](#mistakes)
- [🛡 Защита](#defense)
- [🎓 Чему учит лаборатория](#lessons)
- [✅ Чек-лист](#checklist)

---

<a id="goal"></a>

## 🎯 Цель

Цель лабораторной работы — подобрать пароль пользователя:

```text
carlos
```

и войти в его аккаунт, обойдя защиту от brute-force.

В отличие от предыдущих лабораторий по username enumeration, здесь username жертвы уже известен. Основная задача — понять, почему защита от перебора формально есть, но реализована с логической ошибкой.

Нам также даны собственные учетные данные:

```text
wiener:peter
```

Они нужны не для решения напрямую, а как инструмент обхода защиты.

[⬆ Вернуться к содержанию](#top)

---

<a id="theory"></a>

## 🧠 Краткая теория

**Flawed brute-force protection** — это ситуация, когда приложение пытается защититься от перебора паролей, но делает это неправильно.

Важно понимать: уязвимость не всегда означает, что защиты нет вообще. Иногда защита присутствует, но ее можно обойти из-за ошибки в бизнес-логике.

В этой лаборатории приложение блокирует IP после нескольких неверных попыток входа. На первый взгляд это нормальная защита. Но если пользователь успешно входит в свой аккаунт до достижения лимита, счетчик ошибок сбрасывается.

Это создает возможность чередовать:

```text
успешный вход своим аккаунтом
        ↓
одна попытка подбора пароля Carlos
        ↓
успешный вход своим аккаунтом
        ↓
следующая попытка подбора пароля Carlos
```

Такой подход позволяет не достигать порога блокировки.

[⬆ Вернуться к содержанию](#top)

---

<a id="password-auth"></a>

## 🔐 Password-based Authentication

Password-based authentication — это классический механизм входа по паре:

```text
username + password
```

Сервер обычно выполняет следующие действия:

```text
1. Получает username и password.
2. Находит пользователя в базе.
3. Проверяет пароль.
4. Создает сессию при успешном входе.
5. Возвращает ошибку при неуспешном входе.
```

Упрощенная схема:

```text
POST /login
    ↓
username найден?
    ↓
password правильный?
    ↓
создать session cookie
    ↓
redirect /my-account
```

При неверном пароле приложение должно возвращать ошибку и учитывать неудачную попытку.

Проблема начинается, когда разработчики неправильно решают, **где хранить счетчик ошибок** и **когда его сбрасывать**.

[⬆ Вернуться к содержанию](#top)

---

<a id="bruteforce-types"></a>

## 💣 Brute-force, Credential Stuffing и Password Spraying

### Brute-force

Brute-force — прямой перебор паролей для одного пользователя:

```text
carlos:123456
carlos:password
carlos:qwerty
carlos:football
```

### Credential Stuffing

Credential stuffing — использование уже известных пар `login:password`, утекших из других сервисов:

```text
user@example.com:Password123
admin@example.com:qwerty2024
```

### Password Spraying

Password spraying — попытка одного популярного пароля на множестве аккаунтов:

```text
alice:Winter2024
bob:Winter2024
carlos:Winter2024
admin:Winter2024
```

Эта лаборатория ближе всего к brute-force одного пользователя, но с обходом IP-based защиты.

[⬆ Вернуться к содержанию](#top)

---

<a id="protection"></a>

## 🛡 Как обычно защищаются от brute-force

Типичные способы защиты:

```text
1. IP block
2. Account lockout
3. Rate limiting
4. Progressive delay
5. CAPTCHA
6. MFA
7. Monitoring and alerting
```

### IP block

```text
3 неверные попытки с одного IP
        ↓
IP временно заблокирован
```

### Account lockout

```text
5 неверных попыток для carlos
        ↓
аккаунт carlos временно заблокирован
```

### Progressive delay

```text
1 ошибка  → задержка 1 секунда
2 ошибки → задержка 5 секунд
3 ошибки → задержка 30 секунд
```

### MFA

Даже если пароль подобран, атакующему потребуется второй фактор.

В этой лаборатории используется IP-based блокировка, но она связана с ошибочной логикой сброса счетчика.

[⬆ Вернуться к содержанию](#top)

---

<a id="idea"></a>

## 🧩 Ключевая идея лаборатории

Обычная атака заблокируется:

```text
carlos:123456    ❌
carlos:password  ❌
carlos:qwerty    ❌
        ↓
BLOCK
```

Но если между попытками вставлять успешный вход своим аккаунтом:

```text
wiener:peter     ✅
carlos:123456    ❌
wiener:peter     ✅
carlos:password  ❌
wiener:peter     ✅
carlos:qwerty    ❌
```

то счетчик ошибок постоянно сбрасывается.

Схема:

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

Главная идея:

```text
Мы не ломаем блокировку напрямую.
Мы используем логику приложения против самой защиты.
```

[⬆ Вернуться к содержанию](#top)

---

<a id="logic-flaw"></a>

## 💥 В чем логическая ошибка

Уязвимая логика может выглядеть так:

```python
if login_failed:
    failed_attempts += 1

if login_success:
    failed_attempts = 0

if failed_attempts >= 3:
    block_ip()
```

На первый взгляд это кажется разумным:

```text
если человек успешно вошел,
значит он легитимный,
можно сбросить счетчик ошибок
```

Но проблема в том, что успешный вход может быть выполнен **другим пользователем**.

Например:

```text
Ошибки идут по Carlos,
а сброс делает Wiener.
```

Корректная логика должна быть привязана не только к IP, но и к конкретному аккаунту, который атакуется.

Плохая модель:

```text
failed_attempts[ip]
```

Лучше:

```text
failed_attempts[ip + username]
```

или:

```text
failed_attempts[username]
failed_attempts[ip]
failed_attempts[device/session]
```

[⬆ Вернуться к содержанию](#top)

---

<a id="importance"></a>

## 🧪 Почему эта лаборатория важна

Эта лаборатория учит важной мысли:

> Наличие защиты не означает, что защита реализована правильно.

В реальном пентесте нельзя просто увидеть блокировку и остановиться. Нужно понять:

```text
Когда она срабатывает?
Что именно считается ошибкой?
Что сбрасывает счетчик?
Счетчик общий или отдельный?
Привязан ли он к IP?
Привязан ли он к username?
Можно ли использовать свой аккаунт?
```

Такие баги часто не видны сканерами. Их находят руками через анализ логики приложения.

[⬆ Вернуться к содержанию](#top)

---

<a id="pitchfork"></a>

## 🎯 Почему используется Pitchfork

В Burp Intruder есть разные типы атак. Для этой лаборатории нужен именно **Pitchfork**.

Нам нужно, чтобы два списка шли параллельно:

| Request | Username | Password |
|---:|---|---|
| 1 | wiener | peter |
| 2 | carlos | 123456 |
| 3 | wiener | peter |
| 4 | carlos | password |
| 5 | wiener | peter |
| 6 | carlos | qwerty |

Pitchfork берет:

```text
payload1[1] + payload2[1]
payload1[2] + payload2[2]
payload1[3] + payload2[3]
```

То есть он сохраняет соответствие строк.

[⬆ Вернуться к содержанию](#top)

---

<a id="not-sniper"></a>

## 🚫 Почему не Sniper

Sniper меняет только одну payload-позицию.

Например:

```http
username=carlos&password=§password§
```

Это приведет к обычному перебору:

```text
carlos:123456
carlos:password
carlos:qwerty
```

После нескольких ошибок IP заблокируется.

Sniper не подходит, потому что нам нужно менять **и username, и password** синхронно.

[⬆ Вернуться к содержанию](#top)

---

<a id="not-cluster-bomb"></a>

## 🚫 Почему не Cluster Bomb

Cluster Bomb перебирает все возможные комбинации:

```text
wiener:peter
wiener:123456
wiener:password
carlos:peter
carlos:123456
carlos:password
```

Это ломает нужную последовательность.

Нам не нужны все комбинации. Нам нужен строгий порядок:

```text
wiener:peter
carlos:candidate
wiener:peter
carlos:candidate
```

Поэтому Cluster Bomb не подходит.

[⬆ Вернуться к содержанию](#top)

---

<a id="resource-pool"></a>

## ⚙ Почему нужен Resource Pool = 1

Порядок запросов критически важен.

Если Burp отправит запросы параллельно, сервер может обработать их не в том порядке.

Планируемый порядок:

```text
1. wiener:peter
2. carlos:123456
3. wiener:peter
4. carlos:password
```

Плохой порядок при параллельной отправке:

```text
2. carlos:123456
4. carlos:password
6. carlos:qwerty
1. wiener:peter
```

В таком случае несколько ошибок Carlos могут прийти подряд, и IP будет заблокирован.

Поэтому нужно:

```text
Resource Pool
Maximum concurrent requests = 1
```

Это заставляет Burp отправлять только один запрос за раз.

[⬆ Вернуться к содержанию](#top)

---

<a id="step1"></a>

## 🔍 Шаг 1 — Изучаем страницу логина

Открываем страницу логина и отправляем тестовые данные.

Например:

```text
username=test
password=test
```

В Burp Proxy → HTTP history находим запрос:

```http
POST /login
```

Тело запроса выглядит примерно так:

```http
username=test&password=test
```

Этот запрос будет использоваться как основа для Intruder.

[⬆ Вернуться к содержанию](#top)

---

<a id="step2"></a>

## 🔍 Шаг 2 — Проверяем блокировку после ошибок

Проверяем, как работает защита.

Отправляем несколько неверных попыток:

```text
carlos:test1
carlos:test2
carlos:test3
```

После достижения лимита приложение блокирует IP.

Вывод:

```text
Защита от brute-force есть.
Но нужно проверить ее логику.
```

[⬆ Вернуться к содержанию](#top)

---

<a id="step3"></a>

## 🔍 Шаг 3 — Проверяем сброс счетчика через свой аккаунт

Проверяем последовательность:

```text
carlos:test1  ❌
carlos:test2  ❌
wiener:peter  ✅
carlos:test3  ❌
```

Если после этого блокировки нет, значит успешный вход `wiener:peter` сбрасывает счетчик ошибок.

Именно это и является уязвимостью.

[⬆ Вернуться к содержанию](#top)

---

<a id="step4"></a>

## 🔍 Шаг 4 — Отправляем запрос в Intruder

Отправляем `POST /login` в Intruder.

В Positions ставим две payload-позиции:

```http
username=§wiener§&password=§peter§
```

Две позиции нужны потому, что мы будем менять username и password параллельно.

[⬆ Вернуться к содержанию](#top)

---

<a id="step5"></a>

## 🔍 Шаг 5 — Настраиваем Pitchfork

Выбираем:

```text
Attack type: Pitchfork
```

Это позволит Burp использовать два списка строка к строке.

[⬆ Вернуться к содержанию](#top)

---

<a id="step6"></a>

## 🔍 Шаг 6 — Готовим username payload list

Список username:

```text
wiener
carlos
wiener
carlos
wiener
carlos
...
```

`wiener` должен идти первым, потому что первая строка password list будет `peter`.

Первая пара должна быть успешной:

```text
wiener:peter
```

[⬆ Вернуться к содержанию](#top)

---

<a id="step7"></a>

## 🔍 Шаг 7 — Готовим password payload list

Список password:

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

То есть каждому `wiener` соответствует `peter`, а каждому `carlos` — очередной кандидат из словаря.

Важно:

```text
Количество строк в username list и password list должно совпадать.
```

[⬆ Вернуться к содержанию](#top)

---

<a id="step8"></a>

## 🔍 Шаг 8 — Настраиваем Resource Pool

В Intruder открываем Resource Pool и ставим:

```text
Maximum concurrent requests = 1
```

Это гарантирует, что запросы не перемешаются.

[⬆ Вернуться к содержанию](#top)

---

<a id="step9"></a>

## 🔍 Шаг 9 — Запускаем атаку

Запускаем Intruder.

Ожидаем, что большинство ответов будет:

```http
200 OK
```

Успешный вход определяется по:

```http
302 Found
```

[⬆ Вернуться к содержанию](#top)

---

<a id="step10"></a>

## 🔍 Шаг 10 — Находим пароль Carlos

В результатах находим строку:

```text
username=carlos
status=302
```

Это означает, что пароль найден.

Найденный пароль скрыт:

<details>
<summary>🔑 Показать найденный пароль</summary>

```text
klaster
```

</details>

[⬆ Вернуться к содержанию](#top)

---

<a id="step11"></a>

## 🔍 Шаг 11 — Входим в аккаунт

Входим под:

```text
username=carlos
password=<найденный пароль>
```

Учетные данные скрыты:

<details>
<summary>🔑 Показать найденные учетные данные</summary>

```text
Username: carlos
Password: klaster
```

</details>

После входа в аккаунт лаборатория считается решенной.

[⬆ Вернуться к содержанию](#top)

---

<a id="payloads"></a>

## 🧩 Использованные Payloads

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

Полная логика:

```text
wiener:peter              ✅ сброс
carlos:candidate_1        ❌ проверка
wiener:peter              ✅ сброс
carlos:candidate_2        ❌ проверка
```

[⬆ Вернуться к содержанию](#top)

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

Атака сработала из-за двух условий:

1. Приложение блокирует IP после нескольких ошибок.
2. Успешный вход другого пользователя сбрасывает счетчик ошибок.

В результате атакующий может:

```text
делать попытку для Carlos
        ↓
сбрасывать счетчик через Wiener
        ↓
делать следующую попытку для Carlos
```

Это обход не технической защиты напрямую, а бизнес-логики приложения.

[⬆ Вернуться к содержанию](#top)

---

<a id="real-world"></a>

## 🌍 Практическое применение

Такие ошибки встречаются в реальных приложениях, особенно если:

- rate limiting реализован вручную;
- состояние хранится только на IP;
- счетчик ошибок общий;
- успешный вход очищает слишком много security state;
- отсутствует отдельная логика для атакуемого username.

В bug bounty такие уязвимости могут быть ценными, потому что приводят к реальному account takeover.

[⬆ Вернуться к содержанию](#top)

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

Правильная цепочка мышления:

```text
Есть защита?
    ↓
Да
    ↓
Когда срабатывает?
    ↓
После 3 ошибок
    ↓
Что сбрасывает счетчик?
    ↓
Успешный вход
    ↓
Могу ли я выполнить успешный вход?
    ↓
Да, есть wiener:peter
    ↓
Можно ли чередовать запросы?
    ↓
Да
    ↓
Автоматизируем через Pitchfork
```

Главный вывод:

```text
Пентестер проверяет не только наличие защиты,
но и ее внутреннюю логику.
```

[⬆ Вернуться к содержанию](#top)

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

### 1. Использовать Sniper

Sniper не позволяет чередовать username и password параллельно.

### 2. Использовать Cluster Bomb

Cluster Bomb создает все комбинации и ломает порядок.

### 3. Не настроить Resource Pool

Параллельные запросы могут привести к блокировке.

### 4. Неправильно выровнять списки

Payload lists должны соответствовать строка к строке.

### 5. Начать с Carlos

Первая пара должна быть `wiener:peter`, чтобы стартовать с успешного входа.

### 6. Продолжать атаку после найденного 302

После нахождения правильного пароля лучше остановить атаку.

[⬆ Вернуться к содержанию](#top)

---

<a id="defense"></a>

## 🛡 Защита

Правильная защита должна включать:

- отдельный счетчик ошибок для каждого username;
- отдельный счетчик для IP;
- комбинированный учет `IP + username`;
- progressive delay;
- MFA;
- monitoring;
- detection of login alternation patterns;
- уведомления пользователям о подозрительных попытках.

Нельзя делать так:

```text
любой успешный вход сбрасывает общий счетчик ошибок
```

Лучше:

```text
успешный вход пользователя A
не должен сбрасывать ошибки пользователя B
```

[⬆ Вернуться к содержанию](#top)

---

<a id="lessons"></a>

## 🎓 Чему учит лаборатория

Эта лаборатория учит:

- анализировать логику защиты;
- проверять reset conditions;
- правильно использовать Pitchfork;
- понимать важность порядка запросов;
- использовать Resource Pool;
- отличать наличие защиты от корректности защиты.

[⬆ Вернуться к содержанию](#top)

---

<a id="checklist"></a>

## ✅ Чек-лист

- [x] Проверена блокировка после нескольких ошибок.
- [x] Подтвержден сброс счетчика через `wiener:peter`.
- [x] Найден `POST /login`.
- [x] Запрос отправлен в Intruder.
- [x] Выбран Pitchfork.
- [x] Подготовлен username payload list.
- [x] Подготовлен password payload list.
- [x] Payload lists выровнены.
- [x] Resource Pool установлен в `Maximum concurrent requests = 1`.
- [x] Найден ответ `302 Found`.
- [x] Пароль скрыт в отчете.
- [x] Выполнен вход в аккаунт Carlos.
- [x] Лаборатория решена.

---

# ⬆ Наверх

[Вернуться к содержанию](#top)
