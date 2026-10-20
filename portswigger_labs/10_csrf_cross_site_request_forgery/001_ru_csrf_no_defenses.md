# 📘 PortSwigger Lab: CSRF vulnerability with no defenses

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/learning-paths/csrf/csrf-how-to-construct-a-csrf-attack/csrf/lab-no-defenses
> 🎯 Тема: Cross-Site Request Forgery (CSRF) — базовая эксплуатация без защитных механизмов
> 🧪 Уровень: Apprentice
> ✅ Статус: Solved
> 🛠 Инструменты: Burp Suite (Proxy, Repeater), Exploit Server

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [🗺 Схема атаки](#attack-scheme)
- [🔍 Шаг 1 — Разведка запроса смены email](#step1)
- [🔍 Шаг 2 — Генерация CSRF-пейлоада](#step2)
- [🔍 Шаг 3 — Размещение на Exploit Server](#step3)
- [🔍 Шаг 4 — Проверка и доставка жертве](#step4)
- [📨 Примеры запросов и кода](#requests)
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

Функция смены email в личном кабинете уязвима к CSRF. Задача:

```text
1. Создать HTML-страницу, которая автоматически отправляет запрос на смену email.
2. Разместить её на Exploit Server.
3. Доставить жертве так, чтобы её email изменился.
```

> **Учетные данные для входа в свой аккаунт:** `wiener:peter`.

---

<a id="theory"></a>

## 🧠 Краткая теория

CSRF (Cross-Site Request Forgery) — это атака, при которой злоумышленник заставляет браузер жертвы выполнить нежелательное действие на сайте, где жертва уже аутентифицирована. 

Уязвимость возникает, когда сервер не проверяет, что запрос пришел именно от доверенного источника (например, отсутствует CSRF-токен или проверка заголовка `Referer`/`Origin`). Если браузер автоматически прикрепляет куки сессии к запросу, сервер считает его легитимным.

---

<a id="idea"></a>

## 🧩 Ключевая идея

1. **Отсутствие защиты:** Сервер принимает POST-запрос на смену email без CSRF-токена.
2. **Автоматическая отправка:** HTML-форма с JavaScript (`document.forms[0].submit()`) отправляется сразу при загрузке страницы.
3. **Куки жертвы:** Браузер жертвы автоматически прикрепляет её сессионные куки к запросу, так как домен совпадает.

---

<a id="attack-scheme"></a>

## 🗺 Схема атаки

```text
1. Разведка:     Перехват легитимного POST /my-account/change-email.
2. Пейлоад:      Создание HTML-формы с автоотправкой.
3. Размещение:   Загрузка HTML на Exploit Server.
4. Доставка:     Отправка ссылки жертве (Deliver to victim).
5. Результат:    Email жертвы изменен на указанный в форме.
```

---

<a id="step1"></a>

## 🔍 Шаг 1 — Разведка запроса смены email

1. Войдите в аккаунт `wiener:peter`.
2. В Burp Suite перейдите в **Proxy > HTTP history**.
3. Найдите запрос `POST /my-account/change-email`.
4. Отправьте его в Repeater и изучите структуру. Обычно тело запроса выглядит так: `email=test@test.com`.

---

<a id="step2"></a>

## 🔍 Шаг 2 — Генерация CSRF-пейлоада

**Вариант А (Burp Suite Professional):**
1. В Repeater нажмите правой кнопкой мыши на запрос.
2. Выберите **Engagement tools > Generate CSRF PoC**.
3. Включите опцию **Include auto-submit script** и нажмите **Regenerate**.
4. Скопируйте сгенерированный HTML.

**Вариант Б (Burp Suite Community Edition):**
Используйте следующий шаблон HTML:

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>
    document.forms[0].submit();
</script>
```

* Замените `YOUR-LAB-ID` на идентификатор вашей лабораторной работы.
* Замените `attacker@evil.com` на любой email, который хотите установить (для проверки на себе используйте свой текущий, затем измените).

---

<a id="step3"></a>

## 🔍 Шаг 3 — Размещение на Exploit Server

1. Перейдите на вкладку **Exploit Server** (кнопка вверху страницы лабораторной работы).
2. Вставьте ваш HTML-код в поле **Body**.
3. Нажмите кнопку **Store**.

---

<a id="step4"></a>

## 🔍 Шаг 4 — Проверка и доставка жертве

1. Нажмите **View exploit**, чтобы проверить работу на себе. Проверьте в HTTP history, что запрос прошел и email изменился.
2. **Важно:** измените email в пейлоаде на другой (не тот, что у вас сейчас), чтобы избежать самоблокировки.
3. Нажмите **Deliver to victim**. Лаборатория будет решена.

---

<a id="requests"></a>

## 📨 Примеры запросов и кода

### Легитимный запрос (для анализа)

```http
POST /my-account/change-email HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded

email=test@test.com
```

### CSRF-пейлоад (HTML)

```html
<html>
    <body>
        <form action="https://LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
            <input type="hidden" name="email" value="hacked@evil.com" />
        </form>
        <script>
            document.forms[0].submit();
        </script>
    </body>
</html>
```

---

<a id="response"></a>

## 📥 Пример результата

После доставки жертве, при следующем заходе в аккаунт, её email будет изменен на `hacked@evil.com`. Лабораторная работа помечается как **Solved**.

---

<a id="attack-chain"></a>

## 🧾 Полная цепочка атаки

```text
1. Логин wiener:peter → анализ POST /change-email.
2. Создание HTML с формой и автосабмитом.
3. Загрузка HTML на Exploit Server (Store).
4. Проверка на себе (View exploit).
5. Изменение email в пейлоаде.
6. Deliver to victim → Solved.
```

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

1. **Отсутствие CSRF-токена:** Сервер не проверяет наличие уникального токена в запросе.
2. **Отсутствие проверки Origin/Referer:** Сервер не проверяет, с какого домена пришел запрос.
3. **Автоматическая передача куки:** Браузер жертвы прикрепляет куки `session` к запросу, так как домен совпадает.
4. **Отсутствие SameSite:** Куки передаются при cross-site запросах (в данном случае).

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

```text
1. Ищите state-changing действия: Смена email, пароля, перевод денег.
2. Проверьте токены: Есть ли скрытое поле с токеном? Меняется ли оно?
3. Проверьте заголовки: Есть ли проверка Origin или Referer?
4. Создайте PoC: Используйте Burp (Generate CSRF PoC) или напишите HTML вручную.
5. Тестируйте на себе: Всегда проверяйте пейлоад на своей сессии перед отправкой жертве.
```

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

* **Неверный URL:** Убедитесь, что в `action` указан правильный домен лаборатории.
* **Забыли автосабмит:** Без JavaScript форма не отправится автоматически (жертва должна нажать кнопку).
* **Не изменили email:** Если отправить себе тот же email, можно не заметить изменений или заблокировать аккаунт.
* **Не нажали Store:** Без сохранения на Exploit Server пейлоад не будет доступен.

---

<a id="defense"></a>

## 🛡 Защита

1. **CSRF-токены:** Уникальный, непредсказуемый токен для каждой сессии/формы.
2. **Проверка Origin/Referer:** Отклонять запросы с других доменов.
3. **Атрибут SameSite:** Установить `SameSite=Lax` или `Strict` для куки сессии.
4. **Повторная аутентификация:** Требовать пароль для критических действий.

---

<a id="checklist"></a>

## ✅ Чек-лист

- [ ] Найден запрос `POST /my-account/change-email`
- [ ] Создан HTML-пейлоад с формой и автосабмитом
- [ ] Пейлоад загружен на Exploit Server (Store)
- [ ] Проверен на себе (View exploit)
- [ ] Email изменен на другой
- [ ] Нажата кнопка Deliver to victim
- [ ] Статус: Solved

---

<a id="conclusion"></a>

## 🧾 Итог

Лаборатория решена через базовую CSRF-атаку. Мы использовали отсутствие CSRF-токенов и автоматическую отправку формы для изменения email жертвы. Главный урок: state-changing действия должны быть защищены токенами и проверкой источника запроса.

---

[⬆ Вернуться к началу](#top)
