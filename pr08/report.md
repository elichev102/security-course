# ПР №8. Следы вредоносного ПО в Linux

## 1. Что было посажено

| Механизм | Место | Команда/файл |
|----------|-------|-------------|
| Cron | crontab пользователя | /tmp/.hidden_malware/backdoor.sh |
| Systemd | ~/.config/systemd/user/ | system-helper.service |
| Shell profile | ~/.bashrc | /tmp/.hidden_malware/backdoor.sh |
| Процесс | /tmp/.hidden_malware/ | listener.sh на порту 4444 |

## 2. Что нашли — процессы

**Команда:** `ps aux | grep '/tmp'`
**Результат:** [popa     12345  0.0  0.1  12345  6789 ?  S  20:30  0:00 /bin/bash /tmp/.hidden_malware/backdoor.sh
popa     12346  0.0  0.1  12345  6789 ?  S  20:30  0:00 /bin/bash /tmp/.hidden_malware/listener.sh]
**Что подозрительно:** Процессы запущены из /tmp/.hidden_malware/, что нетипично для системных процессов

## 3. Что нашли — сетевые соединения

**Команда:** `ss -tulnp`
**Подозрительный порт:** 4444
**Процесс:** listener.sh (PID: 15347)

**Команда:** `sudo lsof -i :4444`
**Результат:** [COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nc      15347 popa    3u  IPv4 157349      0t0  TCP *:4444 (LISTEN)]
**Как lsof связывает порт с процессом:** Показывает PID и имя процесса, который слушает порт, а также дескриптор файла

## 4. Что нашли — автозапуск

### Cron
[crontab -l
@reboot /tmp/.hidden_malware/backdoor.sh &
*/5 * * * * /tmp/.hidden_malware/backdoor.sh &]

[Service]
ExecStart=/tmp/.hidden_malware/backdoor.sh
Restart=always
RestartSec=10

[Install]
WantedBy=default.target]
**Что подозрительно:** @reboot и */5 записи с /tmp/.hidden_malware/

### Systemd
[[Unit]
Description=System Helper Service
After=default.target

[Service]
ExecStart=/tmp/.hidden_malware/backdoor.sh
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
]
**Содержимое unit-файла:**
