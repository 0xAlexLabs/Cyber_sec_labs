# 📘 PortSwigger Lab: Username Enumeration via Response Timing

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-response-timing  
> 📄 Candidate usernames: https://portswigger.net/web-security/authentication/auth-lab-usernames  
> 📄 Candidate passwords: https://portswigger.net/web-security/authentication/auth-lab-passwords  
> 🎯 Тема: Authentication vulnerabilities — Username Enumeration через Response Timing + обход IP-блокировки через `X-Forwarded-For`

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [⏱ Почему время ответа раскрывает username](#timing)
- [🌐 Почему нужен X-Forwarded-For](#xff)
- [🎯 Почему используется Pitchfork](#pitchfork)
- [🔍 Шаг 1 — Проверяем обычный login request](#step1)
- [🔍 Шаг 2 — Проверяем блокировку после неверных попыток](#step2)
- [🔍 Шаг 3 — Обходим блокировку через X-Forwarded-For](#step3)
- [🔍 Шаг 4 — Настраиваем Intruder для поиска username](#step4)
- [🔍 Шаг 5 — Находим username по времени ответа](#step5)
- [🔍 Шаг 6 — Подбираем password](#step6)
- [🔍 Шаг 7 — Входим в аккаунт](#step7)
- [🧩 Использованные данные](#payloads)
- [🔬 Почему атака сработала](#breakdown)
- [🧠 Как думать как пентестер](#pentester)
- [❌ Типичные ошибки](#mistakes)
- [🛡 Защита](#defense)
- [✅ Чек-лист](#checklist)

---

<a id="goal"></a>

## 🎯 Цель

Определить валидный username по увеличенному времени ответа сервера, затем подобрать пароль этого пользователя и войти в его аккаунт.

В этой лаборатории нам также выданы тестовые учетные данные:

```text
wiener:peter
```

Они нужны не для решения лаборатории, а как контрольный валидный аккаунт, чтобы сравнить поведение приложения для существующего и несуществующего пользователя.

---

<a id="theory"></a>

## 🧠 Краткая теория

В предыдущих лабораториях Username Enumeration находилась через различия в тексте ответа:

```text
Invalid username
Incorrect password
```

или через почти незаметную разницу:

```text
Invalid username or password.
Invalid username or password 
```

В этой лаборатории текст, статус и длина ответа могут выглядеть одинаково. Основной side-channel — это **время ответа**.

Если username не существует, приложение может быстро вернуть ошибку. Если username существует, приложение дополнительно проверяет пароль, например через медленную функцию хеширования пароля. Поэтому ответ для существующего пользователя может приходить заметно дольше.

---

<a id="idea"></a>

## 🧩 Ключевая идея

Общая логика атаки:

```text
1. Проверить, что приложение блокирует частые неудачные попытки.
2. Найти способ обходить блокировку через X-Forwarded-For.
3. Использовать длинный пароль, чтобы усилить разницу во времени.
4. Перебрать usernames и найти тот, который стабильно отвечает дольше.
5. Зафиксировать найденный username.
6. Перебрать passwords и найти ответ 302 Found.
7. Войти в аккаунт.
```

---

<a id="timing"></a>

## ⏱ Почему время ответа раскрывает username

Уязвимая логика может выглядеть примерно так:

```text
POST /login
    ↓
Найти пользователя в базе
    ↓
Если username не найден:
    сразу вернуть ошибку
    ↓
Если username найден:
    проверить password hash
    вернуть ошибку или выполнить вход
```

Схема:

```text
Invalid username
    ↓
User not found
    ↓
Fast response


Valid username
    ↓
User found
    ↓
Password verification
    ↓
Slower response
```

Если пароль очень длинный, обработка существующего пользователя может стать еще медленнее. Поэтому в лаборатории используется длинная строка пароля примерно на 100 символов.

Пример:

```http
username=testtesttest&password=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

и:

```http
username=wiener&password=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

Для существующего пользователя время ответа обычно выше.

---

<a id="xff"></a>

## 🌐 Почему нужен X-Forwarded-For

Приложение блокирует IP после нескольких неверных попыток входа:

```text
3 неверных попытки
        ↓
IP блокируется на 30 минут
```

Но приложение доверяет заголовку:

```http
X-Forwarded-For
```

Этот заголовок обычно используется прокси-серверами для передачи реального IP клиента backend-приложению. Проблема возникает, если приложение доверяет этому заголовку напрямую от клиента.

Атакующий может отправлять:

```http
X-Forwarded-For: 1
```

```http
X-Forwarded-For: 2
```

```http
X-Forwarded-For: 3
```

Для приложения это выглядит как запросы от разных IP-адресов, хотя все они идут от одного атакующего.

---

<a id="pitchfork"></a>

## 🎯 Почему используется Pitchfork

В Intruder нам нужно менять две позиции одновременно:

```http
X-Forwarded-For: §1§
```

и:

```http
username=§candidate_username§&password=AAAAAAAA...
```

Тип атаки **Pitchfork** берет значения из двух payload lists параллельно:

| Запрос | X-Forwarded-For | Username |
|---:|---:|---|
| 1 | 1 | carlos |
| 2 | 2 | root |
| 3 | 3 | admin |
| 4 | 4 | ads |

Почему не Sniper:

```text
Sniper меняет только одну позицию.
```

Почему не Cluster Bomb:

```text
Cluster Bomb перебирает все комбинации и создает слишком много лишних запросов.
```

Pitchfork здесь оптимален: один username = один новый поддельный IP.

---

<a id="step1"></a>

## 🔍 Шаг 1 — Проверяем обычный login request

Открываем страницу логина и отправляем любые неверные данные:

```text
username=test
password=test
```

В Burp Suite находим запрос:

```http
POST /login
```

Пример тела запроса:

```http
username=test&password=test
```

Отправляем запрос в Repeater:

```text
Right click → Send to Repeater
```

---

<a id="step2"></a>

## 🔍 Шаг 2 — Проверяем блокировку после неверных попыток

В Repeater несколько раз отправляем неверные учетные данные.

После нескольких попыток приложение начинает блокировать вход:

```text
Too many incorrect login attempts
```

или аналогичное сообщение.

В нашем прохождении было замечено:

```text
Каждые 3 неверных запроса появляется блокировка примерно на 30 минут.
```

Это означает, что массовый перебор напрямую не пройдет.

---

<a id="step3"></a>

## 🔍 Шаг 3 — Обходим блокировку через X-Forwarded-For

Добавляем заголовок:

```http
X-Forwarded-For: 1
```

Отправляем неверный запрос несколько раз.

Затем меняем значение:

```http
X-Forwarded-For: 2
```

Если приложение снова принимает запросы, значит оно использует `X-Forwarded-For` как источник IP-адреса.

Это позволяет использовать Intruder для перебора, каждый раз подставляя новое значение заголовка.

---

<a id="step4"></a>

## 🔍 Шаг 4 — Настраиваем Intruder для поиска username

Отправляем запрос в Intruder и выбираем:

```text
Attack type: Pitchfork
```

Добавляем заголовок:

```http
X-Forwarded-For: §1§
```

В теле запроса ставим payload position на username:

```http
username=§test§&password=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

Пароль должен быть длинным, около 100 символов. Это помогает усилить timing difference.

Payload set 1:

```text
Numbers
From: 1
To: 100
Step: 1
Max fraction digits: 0
```

Payload set 2:

```text
Candidate usernames
https://portswigger.net/web-security/authentication/auth-lab-usernames
```

Запускаем атаку.

---

<a id="step5"></a>

## 🔍 Шаг 5 — Находим username по времени ответа

После завершения атаки включаем дополнительные колонки:

```text
Columns → Response received
Columns → Response completed
```

Дальше сортируем результаты по времени ответа.

Большинство usernames дают примерно одинаковое время:

```text
120 ms
140 ms
160 ms
```

Один username отвечает значительно дольше:

```text
1056 ms
```

В нашем прохождении таким username оказался:

<details>
<summary>🔑 Показать найденный username</summary>

```text
ads
```

</details>

Важно: одного измерения недостаточно. Подозрительный username нужно несколько раз проверить в Repeater с длинным паролем и разными значениями `X-Forwarded-For`.

---

<a id="step6"></a>

## 🔍 Шаг 6 — Подбираем password

Создаем новую Intruder-атаку.

Снова используем:

```text
Attack type: Pitchfork
```

Добавляем:

```http
X-Forwarded-For: §1§
```

Username фиксируем найденным значением:

```http
username=ads&password=§test§
```

Payload set 1:

```text
Numbers 1-100
```

Payload set 2:

```text
Candidate passwords
https://portswigger.net/web-security/authentication/auth-lab-passwords
```

Запускаем атаку и ищем ответ:

```http
302 Found
```

Именно `302` означает успешный вход и редирект в личный кабинет.

Найденный password:

<details>
<summary>🔑 Показать найденный password</summary>

```text
soccer
```

</details>

---

<a id="step7"></a>

## 🔍 Шаг 7 — Входим в аккаунт

Найденные учетные данные:

<details>
<summary>🔑 Показать найденные учетные данные</summary>

```text
Username: ads
Password: soccer
```

</details>

Если браузер заблокирован из-за предыдущих неверных попыток, можно войти через Repeater, добавив новый заголовок:

```http
X-Forwarded-For: 1000000
```

и отправить:

```http
username=ads&password=soccer
```

Успешный вход выглядит так:

```http
HTTP/2 302 Found
Location: /my-account?id=ads
Set-Cookie: session=...
```

После этого можно нажать:

```text
Follow redirection
```

Либо выполнить вход через браузер, перехватив запрос в Burp и добавив свежий `X-Forwarded-For`.

---

<a id="payloads"></a>

## 🧩 Использованные данные

Список usernames:

```text
https://portswigger.net/web-security/authentication/auth-lab-usernames
```

Список passwords:

```text
https://portswigger.net/web-security/authentication/auth-lab-passwords
```

Длинный пароль для timing test:

```text
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

Поиск username:

```http
X-Forwarded-For: §1§

username=§candidate_username§&password=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

Подбор password:

```http
X-Forwarded-For: §1§

username=ads&password=§candidate_password§
```

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

В лаборатории было две уязвимости.

### 1. Timing-based Username Enumeration

Приложение обрабатывало существующего пользователя дольше, чем несуществующего.

Причина:

```text
несуществующий username → быстрый отказ
существующий username → дополнительная проверка password hash
```

Это создало side-channel по времени ответа.

### 2. Доверие к X-Forwarded-For

Rate limiting был привязан к IP-адресу, но IP определялся из заголовка, который можно подделать:

```http
X-Forwarded-For: 12345
```

Из-за этого атакующий смог обходить блокировку и выполнять много попыток входа.

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

Правильная логика:

```text
1. Проверить наличие rate limiting.
2. Найти способ его обойти.
3. Проверить, влияет ли username на время ответа.
4. Использовать известный аккаунт wiener:peter как контрольный образец.
5. Усилить timing difference длинным password.
6. Перебирать usernames с новым X-Forwarded-For на каждый запрос.
7. Найти username, который стабильно отвечает дольше.
8. Зафиксировать username и перебрать passwords.
9. Найти 302 Found.
10. Немедленно остановить Intruder после найденного 302.
```

Главный вывод:

```text
Даже если сообщение об ошибке одинаковое, время ответа может раскрывать внутреннюю логику приложения.
```

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

### 1. Делать вывод по одному измерению

Timing атаки шумные. Один медленный ответ может быть случайностью. Подозрительный username нужно перепроверять несколько раз.

### 2. Использовать короткий пароль

Если password короткий, разница во времени может быть слишком маленькой.

### 3. Забыть X-Forwarded-For

Без подмены IP приложение быстро заблокирует перебор.

### 4. Использовать Sniper вместо Pitchfork

Нужно одновременно менять и `X-Forwarded-For`, и username/password.

### 5. Не включить Response received / Response completed

Без этих колонок трудно анализировать timing.

### 6. Не остановить Intruder после 302

Если Intruder продолжит перебор после найденного пароля, он снова отправит много неверных попыток и может заблокировать IP.

---

<a id="defense"></a>

## 🛡 Защита

Правильная защита:

- выполнять одинаковый путь обработки для существующих и несуществующих пользователей;
- использовать одинаковое время ответа или искусственную задержку;
- не раскрывать username через timing side-channel;
- применять rate limiting не только по IP, но и по username, device fingerprint, session и поведению;
- не доверять `X-Forwarded-For` от клиента напрямую;
- принимать `X-Forwarded-For` только от доверенных reverse proxy;
- использовать MFA;
- логировать массовые попытки входа;
- отслеживать аномальное количество разных `X-Forwarded-For`;
- блокировать credential stuffing и password spraying.

Плохая защита:

```text
Одинаковое сообщение об ошибке, но разное время обработки.
```

Еще хуже:

```text
Rate limiting по X-Forwarded-For без проверки доверенного proxy.
```

---

<a id="checklist"></a>

## ✅ Чек-лист

- [x] Найден запрос `POST /login`.
- [x] Проверена блокировка после нескольких неверных попыток.
- [x] Подтвержден обход через `X-Forwarded-For`.
- [x] Использован длинный password для timing test.
- [x] Настроен Intruder Pitchfork.
- [x] Payload 1: numbers для `X-Forwarded-For`.
- [x] Payload 2: candidate usernames.
- [x] Включены `Response received` и `Response completed`.
- [x] Найден username с увеличенным временем ответа.
- [x] Username перепроверен.
- [x] Настроен подбор password.
- [x] Найден ответ `302 Found`.
- [x] Выполнен вход в аккаунт.
- [x] Лаборатория решена.

---

# ⬆ Наверх

[Вернуться к содержанию](#top)
