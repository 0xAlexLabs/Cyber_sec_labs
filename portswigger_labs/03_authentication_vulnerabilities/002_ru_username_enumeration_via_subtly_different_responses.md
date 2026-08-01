# 📘 PortSwigger Lab: Username Enumeration via Subtly Different Responses

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-subtly-different-responses  
> 📄 Candidate usernames: https://portswigger.net/web-security/authentication/auth-lab-usernames  
> 📄 Candidate passwords: https://portswigger.net/web-security/authentication/auth-lab-passwords  
> 🎯 Тема: Authentication vulnerabilities — Username Enumeration через едва заметные различия + Password Brute Force

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [🔍 Шаг 1 — Находим запрос логина](#step1)
- [🔍 Шаг 2 — Настраиваем Intruder](#step2)
- [🔍 Шаг 3 — Настраиваем Grep - Extract](#step3)
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

Найти существующее имя пользователя по едва заметному отличию в сообщении об ошибке, затем подобрать пароль и войти в аккаунт.

---

<a id="theory"></a>

## 🧠 Краткая теория

Эта лаборатория похожа на предыдущую, но отличие в ответах здесь не очевидное.

В прошлой лаборатории приложение явно отвечало разными сообщениями:

```text
Invalid username
Incorrect password
```

Здесь сообщение почти одинаковое:

```text
Invalid username or password.
```

Но для валидного username в ответе есть маленькая ошибка:

```text
Invalid username or password 
```

Разница:

```text
Обычный ответ:       точка в конце
Отличающийся ответ: пробел вместо точки
```

Глазами это почти незаметно, поэтому используется **Grep - Extract** в Burp Intruder.

---

<a id="idea"></a>

## 🧩 Ключевая идея

Сервер старается скрыть Username Enumeration универсальным сообщением:

```text
Invalid username or password.
```

Но разные ветки кода формируют сообщение немного по-разному.

Для несуществующего пользователя:

```html
<p class=is-warning>Invalid username or password.</p>
```

Для существующего пользователя с неверным паролем:

```html
<p class=is-warning>Invalid username or password </p>
```

Именно это отличие позволяет определить валидный username.

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

## 🔍 Шаг 2 — Настраиваем Intruder

На первом этапе перебираем только `username`.

Attack type:

```text
Sniper
```

Payload position:

```http
username=§test§&password=test
```

Пароль оставляем фиксированным.

В Payloads вставляем список:

```text
https://portswigger.net/web-security/authentication/auth-lab-usernames
```

Пока атаку не запускаем — сначала нужно настроить извлечение текста ошибки.

---

<a id="step3"></a>

## 🔍 Шаг 3 — Настраиваем Grep - Extract

Переходим:

```text
Intruder → Settings → Grep - Extract → Add
```

В открывшемся response находим сообщение:

```text
Invalid username or password.
```

Выделяем мышкой только сам текст ошибки, без HTML-тегов:

```text
Invalid username or password.
```

Burp автоматически настраивает границы извлечения. После этого запускаем атаку.

Зачем это нужно:

```text
Grep - Extract выводит текст ошибки в отдельную колонку.
Так проще заметить отличие в одном символе.
```

---

<a id="step4"></a>

## 🔍 Шаг 4 — Находим валидный username

После завершения атаки сортируем результаты по колонке с извлеченным текстом.

Большинство ответов содержит:

```text
Invalid username or password.
```

Один ответ отличается:

```text
Invalid username or password 
```

В HTML это выглядело так:

```html
<p class=is-warning>Invalid username or password </p>
```

В конце сообщения нет точки, вместо нее стоит пробел.

Это означает:

```text
Username существует, но пароль неверный.
```

Найденный username:

<details>
<summary>🔑 Показать найденный username</summary>

```text
pi
```

</details>

---

<a id="step5"></a>

## 🔍 Шаг 5 — Подбираем password

Теперь фиксируем найденный username и перебираем только пароль:

```http
username=pi&password=§test§
```

В Payloads вставляем список:

```text
https://portswigger.net/web-security/authentication/auth-lab-passwords
```

Запускаем атаку.

Успешный пароль определяется по ответу:

```http
302 Found
```

Обычно это означает успешный вход и редирект в личный кабинет.

---

<a id="step6"></a>

## 🔍 Шаг 6 — Входим в аккаунт

Используем найденные учетные данные на странице логина.

<details>
<summary>🔑 Показать найденные учетные данные</summary>

```text
Username: pi
Password: 1111
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
username=pi&password=§candidate_password§
```

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

Разработчик попытался защититься от Username Enumeration, возвращая одинаковое сообщение:

```text
Invalid username or password.
```

Но в реальности сообщения отличались на один символ.

Для несуществующего пользователя:

```text
Invalid username or password.
```

Для существующего пользователя:

```text
Invalid username or password 
```

То есть приложение всё равно выдавало внутреннюю логику:

```text
одна ветка кода → пользователь не найден
другая ветка кода → пользователь найден, пароль неверный
```

Даже если отличие визуально почти незаметно, Burp позволяет его найти через `Grep - Extract`.

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

Правильная логика:

```text
1. Проверить, одинаковы ли ответы на самом деле.
2. Не доверять визуальному просмотру страницы.
3. Извлечь текст ошибки через Grep - Extract.
4. Отсортировать извлеченные значения.
5. Найти ответ, который отличается на один символ.
6. Зафиксировать username.
7. Подобрать password для найденного username.
```

Главная мысль:

```text
Если сообщения выглядят одинаковыми, это не значит, что они действительно одинаковые.
```

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

### 1. Смотреть только глазами

В браузере или обычном Response Viewer отличие в одном пробеле легко пропустить.

### 2. Ориентироваться только на Length

Длина может отличаться из-за других частей страницы. В этой лаборатории надежнее использовать `Grep - Extract`.

### 3. Выделить HTML вместе с сообщением

В Grep - Extract нужно выделять только текст ошибки:

```text
Invalid username or password.
```

а не весь тег `<p>`.

### 4. Сразу запускать Cluster Bomb

Полный перебор сработает, но он менее эффективен и хуже показывает суть лаборатории.

---

<a id="defense"></a>

## 🛡 Защита

Правильная защита:

- использовать полностью одинаковое сообщение для всех ошибок входа;
- формировать ошибку в одном месте кода, а не в разных ветках;
- не допускать отличий в пунктуации, пробелах, длине ответа, статусах и заголовках;
- добавить rate limiting;
- использовать account lockout или progressive delay;
- использовать MFA;
- логировать массовые попытки входа;
- проверять ответы автоматическими тестами на идентичность.

Плохая защита:

```text
Два почти одинаковых сообщения в разных ветках кода.
```

Даже один пробел может стать side-channel.

---

<a id="checklist"></a>

## ✅ Чек-лист

- [x] Найден запрос `POST /login`.
- [x] Запрос отправлен в Intruder.
- [x] Username перебирался отдельно.
- [x] Настроен `Grep - Extract`.
- [x] Извлечено сообщение об ошибке.
- [x] Найден ответ с пробелом вместо точки.
- [x] Определен валидный username.
- [x] Password перебирался отдельно.
- [x] Найден ответ `302 Found`.
- [x] Выполнен вход в аккаунт.
- [x] Лаборатория решена.

---

# ⬆ Наверх

[Вернуться к содержанию](#top)
