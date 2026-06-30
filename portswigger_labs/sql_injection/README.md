# 💉 SQL Injection — PortSwigger Web Security Academy

<p align="center">
  <img src="https://img.shields.io/badge/Platform-PortSwigger-red" />
  <img src="https://img.shields.io/badge/Topic-SQL%20Injection-blue" />
  <img src="https://img.shields.io/badge/Labs-15-success" />
  <img src="https://img.shields.io/badge/Progress-100%25-brightgreen" />
  <img src="https://img.shields.io/badge/Language-RU%20%7C%20EN-purple" />
</p>

<p align="right">
🇬🇧 <a href="./README_EN.md"><b>English Version</b></a>
</p>

> 📂 Путь: `cyber_sec_labs/portswigger_labs/sql_injection`  
> 📚 Раздел: SQL Injection  
> 🧪 Лабораторий: 15  
> 📝 Отчетов: 30 — русская и английская версии для каждой лабораторной

---

## 📚 Содержание

- [О разделе](#-о-разделе)
- [Важное предупреждение](#️-важное-предупреждение)
- [Прогресс](#-прогресс)
- [Roadmap обучения](#️-roadmap-обучения)
- [Карта лабораторных](#-карта-лабораторных)
- [Что изучается](#-что-изучается)
- [Техники SQL Injection](#-техники-sql-injection)
- [СУБД и особенности синтаксиса](#-субд-и-особенности-синтаксиса)
- [Инструменты](#-инструменты)
- [Формат walkthrough](#-формат-walkthrough)
- [Структура каталога](#-структура-каталога)
- [Что дает прохождение раздела](#-что-дает-прохождение-раздела)
- [Защита от SQL Injection](#️-защита-от-sql-injection)
- [Полезные ссылки](#-полезные-ссылки)

---

## 📌 О разделе

Этот каталог содержит разборы лабораторных работ **PortSwigger Web Security Academy** по теме **SQL Injection**.

Цель раздела — не просто собрать готовые решения, а сформировать понятную базу знаний:

- как находить SQL Injection;
- почему конкретный payload работает;
- как выбирать технику атаки под поведение приложения;
- как отличать обычную SQLi от Blind SQLi;
- как адаптировать payload под PostgreSQL, Oracle, MySQL и Microsoft SQL Server;
- как защищать приложение от SQL Injection.

---

## ⚠️ Важное предупреждение

Материалы предназначены **только для обучения**.

Все действия выполнялись исключительно в легальной учебной среде PortSwigger Web Security Academy. Не используйте эти техники против реальных систем без явного письменного разрешения владельца.

---

## 📊 Прогресс

```text
Раздел: SQL Injection
Лабораторий: 15 / 15
Отчетов: 30
Языки: RU / EN
Статус: завершено
```

```text
███████████████ 15 / 15 — 100%
```

---

## 🗺️ Roadmap обучения

```text
SQL Injection
│
├── Basics
│   ├── Hidden Data
│   └── Login Bypass
│
├── UNION Attacks
│   ├── Column Count
│   ├── Text Column
│   ├── Retrieve Data
│   └── Single-Column Concatenation
│
├── Database Enumeration
│   ├── Database Version
│   └── information_schema
│
├── Blind SQL Injection
│   ├── Conditional Responses
│   ├── Conditional Errors
│   ├── Visible Error-Based
│   ├── Time-Based
│   └── OAST
│
├── XML Context
│   └── XML Entity Encoding
│
└── Prevention
    ├── Prepared Statements
    ├── Parameterized Queries
    └── Allow Lists
```

---

## 📋 Карта лабораторных

| № | Лабораторная | Чему учит | RU | EN |
|---|--------------|-----------|:--:|:--:|
| 001 | **Получение скрытых данных через SQL Injection**<br><sub>SQL injection vulnerability in WHERE clause allowing retrieval of hidden data</sub> | WHERE, OR 1=1, SQL-комментарии, извлечение скрытых товаров | [RU](./001_ru_sql_injection_hidden_data.md) | [EN](./001_en_sql_injection_hidden_data.md) |
| 002 | **Обход авторизации через SQL Injection**<br><sub>SQL injection vulnerability allowing login bypass</sub> | обход проверки пароля, комментарии SQL, authentication bypass | [RU](./002_ru_sql_login_bypass.md) | [EN](./002_en_sql_login_bypass.md) |
| 003 | **UNION-атака: определение количества столбцов**<br><sub>SQL injection UNION attack, determining the number of columns returned by the query</sub> | UNION SELECT, NULL, подбор количества колонок | [RU](./003_ru_union_column_count.md) | [EN](./003_en_union_column_count.md) |
| 004 | **UNION-атака: поиск текстовой колонки**<br><sub>SQL injection UNION attack, finding a column containing text</sub> | поиск string-compatible и visible column | [RU](./004_ru_union_text_column.md) | [EN](./004_en_union_text_column.md) |
| 005 | **UNION-атака: получение данных из другой таблицы**<br><sub>SQL injection UNION attack, retrieving data from other tables</sub> | чтение таблицы users, вывод username/password | [RU](./005_ru_union_retrieve_users.md) | [EN](./005_en_union_retrieve_users.md) |
| 006 | **UNION-атака: несколько значений в одной колонке**<br><sub>SQL injection UNION attack, retrieving multiple values in a single column</sub> | конкатенация username и password через разделитель | [RU](./006_ru_union_single_column.md) | [EN](./006_en_union_single_column.md) |
| 007 | **Определение типа и версии базы данных**<br><sub>SQL injection attack, querying the database type and version</sub> | @@version, fingerprinting СУБД, различия синтаксиса | [RU](./007_ru_database_version_mysql_mssql.md) | [EN](./007_en_database_version_mysql_mssql.md) |
| 008 | **Перечисление содержимого базы данных**<br><sub>SQL injection attack, listing the database contents on non-Oracle databases</sub> | information_schema, поиск таблиц и колонок | [RU](./008_ru_db_enumeration_non_oracle.md) | [EN](./008_en_db_enumeration_non_oracle.md) |
| 009 | **Blind SQLi через различие ответов**<br><sub>Blind SQL Injection with Conditional Responses</sub> | TRUE/FALSE канал, Welcome back, извлечение пароля по символам | [RU](./009_ru_blind_sqli_conditional_responses.md) | [EN](./009_en_blind_sqli_conditional_responses.md) |
| 010 | **Blind SQLi через условные ошибки**<br><sub>Blind SQL Injection with Conditional Errors</sub> | Oracle CASE, TO_CHAR(1/0), логика error/no error | [RU](./010_ru_blind_sqli_conditional_errors.md) | [EN](./010_en_blind_sqli_conditional_errors.md) |
| 011 | **Visible Error-Based SQLi**<br><sub>Visible Error-Based SQL Injection</sub> | CAST error, извлечение данных из текста SQL-ошибки | [RU](./011_ru_visible_error_based_sqli.md) | [EN](./011_en_visible_error_based_sqli.md) |
| 012 | **Blind SQLi через задержки и извлечение данных**<br><sub>Blind SQL Injection with Time Delays and Information Retrieval</sub> | pg_sleep(), time-based канал, Intruder, Cluster Bomb | [RU](./012_ru_blind_sqli_time_delays_and_information_retrieval.md) | [EN](./012_en_blind_sqli_time_delays_and_information_retrieval.md) |
| 013 | **Blind SQLi через OAST-взаимодействие**<br><sub>Blind SQL Injection with Out-of-Band Interaction</sub> | Burp Collaborator, DNS callback, подтверждение выполнения SQL | [RU](./013_ru_blind_sqli_oast.md) | [EN](./013_en_blind_sqli_oast.md) |
| 014 | **Blind SQLi через OAST-эксфильтрацию данных**<br><sub>Blind SQL Injection with Out-of-Band Data Exfiltration</sub> | вывод пароля через поддомен Collaborator | [RU](./014_ru_blind_sqli_oast_data_exfiltration.md) | [EN](./014_en_blind_sqli_oast_data_exfiltration.md) |
| 015 | **SQL Injection в XML с обходом WAF**<br><sub>SQL Injection with Filter Bypass via XML Encoding</sub> | XML Entity Encoding, Hackvertor, обход слабого WAF | [RU](./015_ru_sql_injection_filter_bypass_xml_encoding.md) | [EN](./015_en_sql_injection_filter_bypass_xml_encoding.md) |

---

## 🧠 Что изучается

### 1. Базовая SQL Injection

Первые лабораторные объясняют фундамент: пользовательский ввод попадает в SQL-запрос, и атакующий может изменить логику `WHERE`.

Ключевые идеи:

- закрытие строковой кавычки;
- добавление условия `OR 1=1`;
- использование SQL-комментариев;
- обход фильтра `released = 1`;
- обход проверки пароля.

---

### 2. UNION SQL Injection

UNION-атаки позволяют добавить собственный `SELECT` к исходному запросу приложения.

Изучается:

- почему количество колонок должно совпадать;
- зачем используется `NULL`;
- как найти текстовую колонку;
- как вывести данные из таблицы `users`;
- как склеить `username` и `password` в одну строку.

---

### 3. Database Enumeration

После подтверждения SQLi важно понять структуру базы.

Изучается:

- определение типа и версии СУБД;
- чтение `information_schema.tables`;
- чтение `information_schema.columns`;
- поиск нестандартных имен таблиц и колонок;
- извлечение credentials.

---

### 4. Blind SQL Injection

Blind SQLi используется, когда приложение не показывает результат SQL-запроса напрямую.

Каналы извлечения:

- различие текста ответа;
- наличие или отсутствие SQL-ошибки;
- задержка ответа;
- внешний DNS/HTTP callback.

---

### 5. SQL Injection в XML и WAF bypass

Финальная лабораторная показывает, что SQLi может быть не только в URL или cookies, но и внутри XML.

Ключевая цепочка:

```text
HTTP Request → WAF → XML Parser → SQL Engine
```

Слабый WAF видит закодированный payload, а XML Parser декодирует его перед передачей в SQL.

---

## 🧩 Техники SQL Injection

- **Conditional Responses** — логический вывод по различию ответов приложения. Например, сообщение `Welcome back` появляется только при TRUE-условии.
- **Conditional Errors** — логический вывод через намеренно вызванные ошибки SQL. TRUE вызывает ошибку, FALSE возвращает обычную страницу.
- **Visible Error-Based SQL Injection** — получение данных напрямую из текста SQL-ошибки, например через `CAST()`.
- **Time-Based SQL Injection** — извлечение данных по задержкам выполнения запроса, например через `pg_sleep(10)`.
- **Out-of-Band SQL Injection (OAST)** — подтверждение выполнения SQL через внешний DNS/HTTP-запрос.
- **Out-of-Band Data Exfiltration** — передача данных через внешний канал, например пароль в поддомене Burp Collaborator.
- **XML Entity Encoding Bypass** — обход слабого WAF за счет XML-сущностей, которые декодируются уже после проверки.

---

## 🗄 СУБД и особенности синтаксиса

| СУБД | Что изучается |
|------|---------------|
| PostgreSQL | `pg_sleep()`, `CAST()`, `LIMIT`, time-based и visible error-based SQLi |
| Oracle | `dual`, `SUBSTR()`, `TO_CHAR(1/0)`, `xmltype()`, `EXTRACTVALUE()` |
| MySQL | `@@version`, `#` как комментарий, особенности `-- ` |
| Microsoft SQL Server | `@@version`, `WAITFOR DELAY`, OAST через `xp_dirtree` в теории |
| Non-Oracle DB | `information_schema.tables`, `information_schema.columns` |

---

## 🛠 Инструменты

- **Burp Proxy** — перехват запросов.
- **Burp Repeater** — ручная проверка payload'ов.
- **Burp Intruder** — перебор символов и автоматизация blind SQLi.
- **Burp Collaborator** — OAST, DNS/HTTP callbacks.
- **Hackvertor** — кодирование payload'ов, например XML hex entities.
- **Burp Decoder** — URL encoding/decoding.
- **PortSwigger SQL Injection Cheat Sheet** — справочник payload'ов под разные СУБД.

---

## 📝 Формат walkthrough

Каждый разбор построен так, чтобы объяснить не только "что вставить", но и "почему это работает".

Обычно внутри отчета есть:

- цель лаборатории;
- краткая теория;
- пошаговое прохождение;
- использованные payload'ы;
- разбор главного payload;
- объяснение поведения приложения;
- типичные ошибки;
- рекомендации по защите;
- чеклист;
- скрытые credentials через `<details>`.

---

## 📂 Структура каталога

```text
sql_injection/
│
├── README.md
├── README_EN.md
│
├── 001_ru_sql_injection_hidden_data.md
├── 001_en_sql_injection_hidden_data.md
│
├── 002_ru_sql_login_bypass.md
├── 002_en_sql_login_bypass.md
│
├── ...
│
├── 015_ru_sql_injection_filter_bypass_xml_encoding.md
└── 015_en_sql_injection_filter_bypass_xml_encoding.md
```

---

## 🎓 Что дает прохождение раздела

После прохождения раздела вы сможете:

- самостоятельно находить SQL Injection;
- понимать, как backend строит SQL-запрос;
- определять тип СУБД;
- строить payload'ы под PostgreSQL, Oracle, MySQL и MSSQL;
- использовать UNION SQL Injection;
- эксплуатировать Blind SQLi через ответы, ошибки, задержки и OAST;
- извлекать данные из других таблиц;
- обходить слабые WAF;
- объяснить, почему Prepared Statements защищают от SQL Injection.

---

## 🛡️ Защита от SQL Injection

Главная защита:

```text
Prepared Statements / Parameterized Queries
```

Дополнительно:

- не собирать SQL через конкатенацию строк;
- использовать allow-list для `ORDER BY`, имен таблиц и колонок;
- валидировать типы данных;
- ограничивать права database user;
- скрывать подробные SQL-ошибки;
- мониторить подозрительные запросы;
- ограничивать исходящие DNS/HTTP-запросы от серверов БД;
- не полагаться только на WAF.

---

## 🔗 Полезные ссылки

- PortSwigger SQL Injection: https://portswigger.net/web-security/sql-injection
- SQL Injection Cheat Sheet: https://portswigger.net/web-security/sql-injection/cheat-sheet
- Burp Suite: https://portswigger.net/burp

---

## 🚀 Что дальше

После SQL Injection логично продолжать:

- Cross-Site Scripting (XSS);
- Authentication;
- Access Control;
- XXE;
- SSRF.

---

⬆ [Наверх](#-sql-injection--portswigger-web-security-academy)
