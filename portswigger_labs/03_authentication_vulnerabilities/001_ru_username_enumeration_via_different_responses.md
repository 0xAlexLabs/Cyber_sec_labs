# 📘 PortSwigger Lab: Username Enumeration via Different Responses

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/authentication/lab-username-enumeration-via-different-responses  
> 📄 Candidate usernames: https://portswigger.net/web-security/authentication/auth-lab-usernames  
> 📄 Candidate passwords: https://portswigger.net/web-security/authentication/auth-lab-passwords  
> 🎯 Тема: Authentication vulnerabilities — Username Enumeration + Password Brute Force

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [🔍 Шаг 1 — Находим запрос логина](#step1)
- [🔍 Шаг 2 — Перебираем username](#step2)
- [🔍 Шаг 3 — Находим валидный username](#step3)
- [🔍 Шаг 4 — Перебираем password](#step4)
- [🔍 Шаг 5 — Входим в аккаунт](#step5)
- [🧩 Использованные данные](#payloads)
- [🔬 Почему атака сработала](#breakdown)
- [🧠 Как думать как пентестер](#pentester)
- [❌ Типичные ошибки](#mistakes)
- [🛡 Защита](#defense)
- [✅ Чек-лист](#checklist)

---

<a id="goal"></a>

## 🎯 Цель

Найти существующее имя пользователя по различию ответов сервера, затем подобрать пароль для этого пользователя и войти в его аккаунт.

---

<a id="theory"></a>

## 🧠 Краткая теория

**Username Enumeration** — это ситуация, когда приложение по-разному отвечает на попытки входа с несуществующим и существующим пользователем.

Например:

```text
Invalid username
```

и:

```text
Incorrect password
```

Для обычного пользователя это просто сообщения об ошибке, а для атакующего — подсказка:

```text
Если ответ изменился, значит username существует.
```

После нахождения валидного username атакующему уже не нужно перебирать все пары `username:password`. Можно перебирать только пароли для одного найденного пользователя.

---

<a id="idea"></a>

## 🧩 Ключевая идея

В лаборатории есть два этапа:

```text
1. Перебрать username с фиксированным password.
2. Перебрать password для найденного username.
```

Это эффективнее, чем сразу делать полный перебор всех комбинаций.

```text
100 usernames × 100 passwords = 10 000 запросов
100 usernames + 100 passwords = 200 запросов
```

---

<a id="step1"></a>

## 🔍 Шаг 1 — Находим запрос логина

Открываем страницу логина и отправляем любые неверные данные:

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

Отправляем его в Intruder:

```text
Right click → Send to Intruder
```

---

<a id="step2"></a>

## 🔍 Шаг 2 — Перебираем username

В Intruder лучше сначала использовать **Sniper**, а не Cluster Bomb.

Ставим payload position только на параметр `username`:

```http
username=§test§&password=test
```

Пароль оставляем фиксированным, например:

```text
test
```

В Payloads вставляем список из PortSwigger:

```text
Candidate usernames
https://portswigger.net/web-security/authentication/auth-lab-usernames
```

Запускаем атаку.

---

<a id="step3"></a>

## 🔍 Шаг 3 — Находим валидный username

После завершения атаки смотрим на ответы в Intruder.

Большинство ответов одинаковые:

```text
Invalid username
```

Но один ответ отличается. В этой лаборатории отличие можно заметить по:

```text
Length
```

или по тексту ответа:

```text
Incorrect password
```

Это означает:

```text
Username существует, но пароль неверный.
```

Найденный username:

<details>
<summary>🔑 Показать найденный username</summary>

```text
argentina
```

</details>

---

<a id="step4"></a>

## 🔍 Шаг 4 — Перебираем password

Теперь меняем запрос.

Username фиксируем найденным значением:

```http
username=argentina&password=§test§
```

Payload position ставим только на `password`.

В Payloads вставляем список паролей:

```text
Candidate passwords
https://portswigger.net/web-security/authentication/auth-lab-passwords
```

Запускаем атаку.

На этот раз ищем не отличие по `Length`, а успешный вход. Обычно он проявляется как:

```http
302 Found
```

или редирект на:

```text
/my-account
```

Все неверные пароли обычно возвращают:

```http
200 OK
```

---

<a id="step5"></a>

## 🔍 Шаг 5 — Входим в аккаунт

После нахождения правильного пароля входим через форму логина.

<details>
<summary>🔑 Показать найденные учетные данные</summary>

```text
Username: argentina
Password: 111111
```

</details>

После входа в аккаунт лаборатория считается решенной.

---

<a id="payloads"></a>

## 🧩 Использованные данные

Список username:

```text
https://portswigger.net/web-security/authentication/auth-lab-usernames
```

Список password:

```text
https://portswigger.net/web-security/authentication/auth-lab-passwords
```

Первая атака:

```http
username=§candidate_username§&password=test
```

Вторая атака:

```http
username=argentina&password=§candidate_password§
```

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

Приложение выдавало разные сообщения для разных ситуаций.

Для несуществующего пользователя:

```text
Invalid username
```

Для существующего пользователя с неверным паролем:

```text
Incorrect password
```

Это создало side-channel: приложение прямо не показывает список пользователей, но через разницу в ответах позволяет его восстановить.

После нахождения валидного username задача сильно упрощается: вместо перебора всех комбинаций остается подобрать только password для одного пользователя.

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

Правильная логика:

```text
1. Не начинать сразу с полного brute force.
2. Проверить, отличаются ли ответы при неверном username и неверном password.
3. Если отличаются — сначала выполнить username enumeration.
4. Затем подобрать password только для найденного username.
5. Успешный вход определить по 302 redirect или переходу в /my-account.
```

Главный навык этой лаборатории — замечать маленькие различия:

```text
разный текст ошибки
разная длина ответа
разный status code
разный redirect
```

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

### 1. Сразу запускать Cluster Bomb

Это сработает, но очень неэффективно:

```text
100 × 100 = 10 000 запросов
```

Правильнее сначала найти username:

```text
100 + 100 = 200 запросов
```

### 2. Искать только status code на первом этапе

На этапе username enumeration status code часто одинаковый:

```http
200 OK
```

Нужно смотреть на `Length` и текст ответа.

### 3. Искать только Length на втором этапе

На этапе password brute force важнее status code:

```http
302 Found
```

Это признак успешного входа.

---

<a id="defense"></a>

## 🛡 Защита

Правильная защита:

- возвращать одинаковое сообщение для неверного username и неверного password;
- использовать единый ответ, например `Invalid username or password`;
- добавить rate limiting;
- добавить account lockout или задержки после неудачных попыток;
- использовать MFA;
- логировать подозрительные попытки входа;
- отслеживать массовые переборы username/password;
- не раскрывать существование аккаунта через текст, длину ответа или разные статусы.

Плохой вариант:

```text
Invalid username
Incorrect password
```

Такие сообщения помогают атакующему перечислять пользователей.

---

<a id="checklist"></a>

## ✅ Чек-лист

- [x] Найден запрос `POST /login`.
- [x] Запрос отправлен в Intruder.
- [x] Username перебирался отдельно.
- [x] Найден отличающийся ответ.
- [x] Определен валидный username.
- [x] Password перебирался отдельно.
- [x] Найден ответ `302 Found`.
- [x] Выполнен вход в аккаунт.
- [x] Лаборатория решена.

---

# ⬆ Наверх

[Вернуться к содержанию](#top)
