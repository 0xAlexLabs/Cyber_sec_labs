# 📘 PortSwigger Lab: Username Enumeration via Account Lock

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-account-lock  
> 📄 Candidate usernames: https://portswigger.net/web-security/authentication/auth-lab-usernames  
> 📄 Candidate passwords: https://portswigger.net/web-security/authentication/auth-lab-passwords  
> 🎯 Тема: Authentication vulnerabilities — Username Enumeration через Account Lock + Password Brute Force

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [🔒 Почему `test:test` не блокируется](#test-test)
- [🎯 Почему используется Cluster Bomb](#cluster-bomb)
- [⚙️ Почему используется Null Payload](#null-payload)
- [🔍 Шаг 1 — Находим запрос логина](#step1)
- [🔍 Шаг 2 — Проверяем обычную ошибку входа](#step2)
- [🔍 Шаг 3 — Настраиваем Intruder для username enumeration](#step3)
- [🔍 Шаг 4 — Находим валидный username](#step4)
- [🔍 Шаг 5 — Подбираем password](#step5)
- [🔍 Шаг 6 — Входим в аккаунт](#step6)
- [🧩 Использованные данные](#payloads)
- [🔬 Почему атака сработала](#breakdown)
- [🧠 Как думать как пентестер](#pentester)
- [❌ Типичные ошибки](#mistakes)
- [🛡 Защита](#defense)
- [✅ Чек-лист](#checklist)

---

<a id="goal"></a>

## 🎯 Цель

Найти существующее имя пользователя через механизм блокировки аккаунта, затем подобрать пароль этого пользователя и войти в его аккаунт.

В этой лаборатории приложение использует **account locking**: после нескольких неверных попыток входа аккаунт временно блокируется. На первый взгляд это защита от brute-force, но из-за особенностей реализации она раскрывает существующие usernames.

---

<a id="theory"></a>

## 🧠 Краткая теория

**Account Locking** — это механизм защиты, при котором приложение временно блокирует аккаунт после нескольких неверных попыток входа.

Например:

```text
ftp:wrong1  ❌
ftp:wrong2  ❌
ftp:wrong3  ❌
ftp:wrong4  ❌
ftp:wrong5  ❌
        ↓
Account locked
```

Такая защита полезна против перебора одного конкретного аккаунта. Но она может стать источником Username Enumeration, если приложение блокирует только реально существующие аккаунты.

В этой лаборатории обычная ошибка выглядит так:

```text
Invalid username or password.
```

А для существующего пользователя после нескольких неверных попыток появляется другое сообщение:

```text
You have made too many incorrect login attempts. Please try again in 1 minute(s).
```

Это сообщение раскрывает, что username существует.

---

<a id="idea"></a>

## 🧩 Ключевая идея

Для несуществующего пользователя сервер всегда возвращает обычную ошибку:

```text
username=test
password=test
        ↓
Invalid username or password.
```

Даже если отправить этот запрос много раз подряд, блокировка не появляется, потому что аккаунта `test` нет.

Для существующего пользователя поведение другое:

```text
username=ftp
password=example
        ↓
failed_attempts++
        ↓
после нескольких попыток
        ↓
You have made too many incorrect login attempts.
```

Схема:

```text
Несуществующий username
        │
        ├─ user not found
        │
        └─ Invalid username or password.


Существующий username
        │
        ├─ user found
        ├─ password wrong
        ├─ failed_attempts++
        └─ Account locked
```

Именно это различие позволяет найти валидный username.

---

<a id="test-test"></a>

## 🔒 Почему `test:test` не блокируется

Во время ручной проверки можно заметить, что запрос:

```text
username=test
password=test
```

всегда возвращает:

```text
Invalid username or password.
```

Даже после пяти попыток сообщение не меняется.

Это важное наблюдение. Оно означает, что приложение не увеличивает счетчик ошибок для несуществующего пользователя.

Упрощенная логика сервера может выглядеть так:

```python
user = find_user(username)

if not user:
    return "Invalid username or password."

if password_is_wrong:
    user.failed_attempts += 1

if user.failed_attempts >= 5:
    return "You have made too many incorrect login attempts."
```

Проблема здесь в том, что ветка `not user` завершается раньше, чем приложение доходит до счетчика ошибок.

---

<a id="cluster-bomb"></a>

## 🎯 Почему используется Cluster Bomb

На первом этапе нам нужно не подобрать пароль, а заставить приложение несколько раз проверить каждый username.

Если использовать обычный Sniper, получится:

```text
carlos:example
root:example
admin:example
test:example
```

Каждый username будет проверен только один раз. Этого недостаточно, чтобы вызвать account lock.

Нам нужно так:

```text
carlos:example
carlos:example
carlos:example
carlos:example
carlos:example

root:example
root:example
root:example
root:example
root:example
```

Поэтому используется **Cluster Bomb** с двумя payload positions:

```http
username=§test§&password=example§§
```

Первый payload меняет username. Второй payload пустой, но заставляет Burp повторять каждый username несколько раз.

---

<a id="null-payload"></a>

## ⚙️ Почему используется Null Payload

`Null Payload` не изменяет запрос. Он подставляет пустую строку.

Если тело запроса выглядит так:

```http
username=§test§&password=example§§
```

то password остается:

```text
example
```

Но Cluster Bomb создает комбинации:

```text
Payload 1 × Payload 2
```

Если в Payload 1 — 101 username, а в Payload 2 — 5 пустых payloads, получится:

```text
101 × 5 = 505 requests
```

То есть каждый username будет автоматически отправлен пять раз.

---

<a id="step1"></a>

## 🔍 Шаг 1 — Находим запрос логина

Открываем страницу логина и вводим любые неверные данные:

```text
username=test
password=test
```

В Burp Suite переходим:

```text
Proxy → HTTP history
```

Находим запрос:

```http
POST /login
```

Пример тела запроса:

```http
username=test&password=test
```

Отправляем запрос в Intruder:

```text
Right click → Send to Intruder
```

---

<a id="step2"></a>

## 🔍 Шаг 2 — Проверяем обычную ошибку входа

Базовый ответ приложения:

```http
HTTP/2 200 OK
Content-Length: 3236
```

Сообщение в HTML:

```html
<p class=is-warning>Invalid username or password.</p>
```

Это эталонный ответ. Дальше мы будем искать ответы, которые отличаются от него по длине или тексту ошибки.

---

<a id="step3"></a>

## 🔍 Шаг 3 — Настраиваем Intruder для username enumeration

В Intruder выбираем:

```text
Attack type: Cluster Bomb
```

Payload positions:

```http
username=§test§&password=example§§
```

Настройка payloads:

```text
Payload 1:
Simple list → candidate usernames

Payload 2:
Null payloads → generate 5 payloads
```

После запуска Burp отправит каждый username пять раз с одним и тем же неверным паролем `example`.

---

<a id="step4"></a>

## 🔍 Шаг 4 — Находим валидный username

После завершения атаки сортируем ответы по `Length` и ищем отличающиеся ответы.

Для большинства usernames длина ответа одинаковая и сообщение остается:

```text
Invalid username or password.
```

Для существующего username появляется другое сообщение:

```html
<p class=is-warning>You have made too many incorrect login attempts. Please try again in 1 minute(s).</p>
```

В нашем прохождении отличающийся username был:

<details>
<summary>🔑 Показать найденный username</summary>

```text
ftp
```

</details>

Это означает:

```text
username существует
        ↓
сервер начал считать failed attempts
        ↓
аккаунт был временно заблокирован
```

---

<a id="step5"></a>

## 🔍 Шаг 5 — Подбираем password

После нахождения username нужно подождать примерно минуту, чтобы блокировка аккаунта сбросилась.

Затем создаем новую Intruder-атаку:

```text
Attack type: Sniper
```

Фиксируем username и меняем только password:

```http
username=ftp&password=§candidate_password§
```

В Payloads вставляем список паролей:

```text
https://portswigger.net/web-security/authentication/auth-lab-passwords
```

Успешный пароль определяется по ответу:

```http
302 Found
```

В нашем прохождении правильный пароль дал редирект:

```http
HTTP/2 302 Found
```

---

<a id="step6"></a>

## 🔍 Шаг 6 — Входим в аккаунт

Используем найденные учетные данные на странице логина.

<details>
<summary>🔑 Показать найденные учетные данные</summary>

```text
Username: ftp
Password: 12345678
```

</details>

После входа в аккаунт и открытия страницы пользователя лаборатория считается решенной.

---

<a id="payloads"></a>

## 🧩 Использованные данные

Username list:

```text
https://portswigger.net/web-security/authentication/auth-lab-usernames
```

Password list:

```text
https://portswigger.net/web-security/authentication/auth-lab-passwords
```

Первая атака:

```http
username=§candidate_username§&password=example§§
```

Payload 2:

```text
Null payloads × 5
```

Вторая атака:

```http
username=ftp&password=§candidate_password§
```

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

Разработчик хотел защитить приложение от brute-force с помощью account lockout.

Но логика была реализована так, что счетчик ошибок срабатывал только для существующих пользователей.

Для несуществующего пользователя:

```text
user not found
        ↓
Invalid username or password.
```

Для существующего пользователя:

```text
user found
        ↓
password wrong
        ↓
failed_attempts++
        ↓
account locked
```

Это создало side-channel: сообщение о блокировке стало признаком существования username.

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

Правильная логика:

```text
1. Проверить обычную ошибку входа.
2. Проверить, появляется ли блокировка.
3. Понять, что несуществующий username не блокируется.
4. Повторить каждый username несколько раз.
5. Найти username, для которого появляется account lock.
6. Дождаться сброса блокировки.
7. Подобрать password только для найденного username.
```

Главная мысль:

```text
Механизм защиты может сам стать источником утечки информации.
```

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

### 1. Сразу подбирать пароль

Если username неизвестен, перебор паролей будет бессмысленным.

### 2. Использовать Sniper на первом этапе

Sniper проверит каждый username только один раз и не вызовет account lock.

### 3. Неправильно настроить Null Payload

Второй payload должен быть пустым и использоваться только для повторения запросов.

### 4. Смотреть только на Status Code

В первой атаке статус обычно остается `200 OK`. Важнее текст ответа и Length.

### 5. Не ждать снятия блокировки

Перед подбором пароля нужно подождать, иначе все ответы будут содержать сообщение о блокировке.

---

<a id="defense"></a>

## 🛡 Защита

Правильная защита:

- возвращать одинаковое сообщение для существующих и несуществующих пользователей;
- не раскрывать факт блокировки аккаунта до успешной идентификации пользователя безопасным способом;
- применять rate limiting по IP, username и комбинации IP + username;
- использовать progressive delay;
- использовать MFA;
- логировать массовые попытки входа;
- обнаруживать password spraying и credential stuffing;
- проверять, что ответы аутентификации не раскрывают внутренние ветки логики.

Плохая защита:

```text
Invalid username or password.
```

для несуществующего пользователя и:

```text
You have made too many incorrect login attempts.
```

для существующего пользователя.

---

<a id="checklist"></a>

## ✅ Чек-лист

- [x] Найден запрос `POST /login`.
- [x] Проверен обычный ответ `Invalid username or password.`
- [x] Подтверждено, что несуществующий username не блокируется.
- [x] Настроен `Cluster Bomb`.
- [x] Настроен `Null Payload × 5`.
- [x] Найден ответ с `You have made too many incorrect login attempts`.
- [x] Определен валидный username.
- [x] Дождались снятия блокировки.
- [x] Настроен `Sniper` для password brute force.
- [x] Найден ответ `302 Found`.
- [x] Выполнен вход в аккаунт.
- [x] Лаборатория решена.

---

# ⬆ Наверх

[Вернуться к содержанию](#top)
