# ПР №7. AppArmor, Capabilities и Docker

## 1. Linux Capabilities

### Разбор getcap /usr/bin/ping
**Что означают поля:**
- **cap_net_raw** — разрешение на создание RAW-сокетов (нужно для ICMP/ping)
- **e** (effective) — capability активна и используется процессом
- **p** (permitted) — процесс может использовать эту capability

### CapPrm / CapEff / CapBnd — в чём разница
- **CapPrm** (Permitted) — capabilities, которые процесс может использовать
- **CapEff** (Effective) — capabilities, которые активны в данный момент
- **CapBnd** (Bounding) — максимальный набор capabilities, который процесс может получить

### setcap — демонстрация
**До:** `python3 /tmp/test-port.py 80` → `DENIED: порт 80 — Permission denied`

**После** `setcap cap_net_bind_service=ep` → `OK: привязался к порту 80`

**Почему лучше чем sudo:** setcap даёт процессу только одно конкретное право (привязку к порту), а не полный root-доступ. Это принцип наименьших привилегий.

### Флаги e, i, p в cap_net_raw+eip
- **e** (effective) — capability активна
- **i** (inheritable) — может быть унаследована дочерними процессами
- **p** (permitted) — процесс может использовать эту capability

---

## 2. AppArmor

### Количество профилей
- **enforce:** 3
- **complain:** 0

### Результаты pr07-reader

| Действие | Без профиля | complain | enforce |
|---|---|---|---|
| Читать /tmp/pr07-allowed.txt | ✅ Успех | ✅ Успех | ❌ Отказ |
| Читать /etc/shadow | ❌ Отказ | ❌ Отказ | ❌ Отказ |
| Писать в /tmp/pr07-output.txt | ✅ Успех | ✅ Успех | ❌ Отказ |
| Писать в /etc/pr07-hack.txt | ❌ Отказ | ❌ Отказ | ❌ Отказ |

### Разбор строки DENIED
| Поле | Значение |
|---|---|
| operation | Тип операции (open, exec, connect) |
| profile | Профиль AppArmor, который заблокировал действие |
| name | Путь к файлу, к которому был запрещён доступ |
| denied_mask | Какое право запрошено и отклонено (r — чтение, w — запись, x — выполнение) |

---

## 3. Docker — изоляция

### Сравнение хоста и контейнера

| Ресурс | Хост | Контейнер |
|---|---|---|
| Количество процессов | ~220 шт | 1 шт |
| Сетевые интерфейсы | eth0, lo, docker0 | lo, eth0 |
| /etc/shadow хоста | доступен | недоступен |
| Монтирование | разрешено | запрещено |

### Capabilities: обычный vs --privileged

**Обычный CapEff:** `00000000a80425fb`
→ `cap_chown, cap_dac_override, cap_fowner, cap_fsetid, cap_kill, cap_setgid, cap_setuid, cap_setpcap, cap_net_bind_service, cap_net_raw, cap_sys_chroot, cap_mknod, cap_audit_write, cap_setfcap`

**--privileged CapEff:** `00000003ffffffff`
→ почти все возможные capabilities (включая sys_admin, sys_rawio и другие)

**Чего нет у обычного контейнера:**
- `cap_sys_admin` — монтирование, изменение ядра
- `cap_sys_rawio` — прямой доступ к диску
- `cap_sys_ptrace` — отладка процессов
- `cap_net_admin` — настройка сети

**Почему --privileged опасен:** Даёт контейнеру почти все права root на хосте — может монтировать диски, менять сетевые настройки, загружать модули ядра.

---

### Итоговый nginx

**Запущен с capabilities:**
- `NET_BIND_SERVICE` — привязка к порту 80
- `CHOWN` — изменение владельца файлов
- `DAC_OVERRIDE` — обход прав доступа
- `SETGID/SETUID` — смена пользователя

**Почему именно эти:** nginx работает с файлами и привязывается к порту 80, но не нуждается в полном root-доступе. Минимально необходимый набор.

---

## 4. Эшелонированная защита

| Слой | Инструмент | Что ограничивает |
|---|---|---|
| DAC | chmod/chown | Права доступа к файлам |
| Capabilities | --cap-drop ALL + cap-add | Права процессов (системные вызовы) |
| MAC | AppArmor | Доступ к файлам и операциям |
| Изоляция | Docker namespaces | Видимость процессов, сети, ФС |

---

## Выводы

В ходе работы изучены три уровня защиты:

1. **Linux Capabilities** — позволяют дать процессу только необходимые права вместо полного root
2. **AppArmor** — мандатный контроль доступа, ограничивающий файлы и операции
3. **Docker** — изоляция через namespaces и cgroups

Комбинация этих слоёв создаёт **эшелонированную защиту**, которая значительно усложняет действия злоумышленника даже при наличии уязвимости в приложении.
