# ПР №7. AppArmor, Capabilities и Docker

## 1. Linux Capabilities

### Разбор getcap /usr/bin/ping

**Вывод:** `cap_net_raw=ep`

| Часть | Значение |
|-------|----------|
| cap_net_raw | Право на raw-сокеты (нужен для ICMP/ping) |
| e (effective) | Capability активна сразу |
| p (permitted) | Процесс может использовать эту capability |

### CapPrm / CapEff / CapBnd

| Поле | Что означает |
|------|--------------|
| CapPrm | Permitted - разрешённые capabilities |
| CapEff | Effective - активные прямо сейчас |
| CapBnd | Bounding - максимально возможные |

### Демонстрация setcap

| Состояние | Результат |
|-----------|-----------|
| Без setcap | DENIED: порт 80 (Permission denied) |
| После `setcap cap_net_bind_service=ep` | OK: порт 80 |

**Почему лучше чем sudo:** Даётся ТОЛЬКО одно право (bind на порт 80), а не полный root.

### Флаги e, i, p в cap_net_raw+eip

| Флаг | Значение |
|------|----------|
| e (effective) | Активна сразу |
| i (inheritable) | Может передаваться дочерним процессам |
| p (permitted) | Процесс может использовать |

## 2. AppArmor

### Количество профилей

- enforce: (посмотреть в `sudo aa-status`)
- complain: (посмотреть в `sudo aa-status`)

### Результаты pr07-reader

| Действие | Без профиля | complain | enforce |
|----------|-------------|----------|---------|
| Читать /tmp/pr07-allowed.txt | OK | OK | OK |
| Читать /etc/shadow | OK | OK (лог) | DENIED |
| Писать в /tmp/pr07-output.txt | OK | OK | OK |
| Писать в /etc/ | OK | OK (лог) | DENIED |

### Разбор строки DENIED
[ 123.456] audit: type=1400 audit(123.456:789): apparmor="DENIED" operation="open" profile="/usr/local/bin/pr07-reader" name="/etc/shadow" pid=1234 comm="cat" requested_mask="r" denied_mask="r" fsuid=0 ouid=0

| Поле | Значение |
|------|----------|
| operation="open" | Попытка открыть файл |
| profile="pr07-reader" | Какой профиль заблокировал |
| name="/etc/shadow" | Какой файл пытались открыть |
| requested_mask="r" | Какое право запрашивали |
| comm="cat" | Какая команда пыталась |

## 3. Docker изоляция

### Сравнение хоста и контейнера

| Ресурс | Хост | Контейнер |
|--------|------|-----------|
| Количество процессов | ~150 | ~1-2 |
| Сетевые интерфейсы | eth0, lo, docker0 | lo, eth0 |
| /etc/shadow хоста | доступен | **НЕ ДОСТУПЕН** |
| Монтирование ФС | разрешено | **ЗАПРЕЩЕНО** |

### Capabilities: обычный vs --privileged

| Тип | CapEff (hex) | Расшифровка |
|-----|--------------|-------------|
| Обычный контейнер | (значение) | cap_chown, cap_dac_override, cap_fowner, cap_setgid, cap_setuid, cap_net_bind_service... |
| --privileged | (значение) | **ВСЕ capabilities** (почти как root на хосте) |

**Чего нет у обычного контейнера:** cap_sys_admin, cap_sys_rawio, cap_sys_module, cap_sys_ptrace, cap_net_raw

**Почему --privileged опасен:** Даёт контейнеру полный доступ к ядру хоста - контейнер может выйти на хост.

### Запуск не от root

| Команда | Результат |
|---------|-----------|
| `docker run ubuntu whoami` | root |
| `docker run --user 1000 ubuntu whoami` | 1000 |
| `docker run --user 1000 ubuntu apt update` | DENIED (нет прав) |

**Почему важно:** Если взломают процесс от root - получат полный контроль. Если от непривилегированного пользователя - ущерб ограничен.

## 4. Итоговый nginx с ограничениями

```bash
docker run -d --name pr07-nginx \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --cap-add CHOWN \
  --cap-add DAC_OVERRIDE \
  --cap-add SETGID \
  --cap-add SETUID \
  -p 8080:80 nginx:alpine
Capabilities nginx: cap_chown, cap_dac_override, cap_net_bind_service, cap_setgid, cap_setuid

Почему именно эти:

NET_BIND_SERVICE - слушать порт 80

CHOWN - менять владельца файлов

DAC_OVERRIDE - обходить права файлов

SETGID/SETUID - менять ID процессов

5. Эшелонированная защита (Приказ №17)
Слой	Инструмент	Что ограничивает
DAC	chmod/chown	Доступ пользователей к файлам
Capabilities	--cap-drop/add	Права процессов (принцип наименьших)
MAC	AppArmor	Доступ программ к файлам/сети
Изоляция	Docker namespaces	Видимость процессов, сети, ФС
Выводы
Научился:

Выдавать и удалять capabilities через setcap

Создавать профили AppArmor в режимах complain/enforce

Изолировать процессы через Docker namespaces

Ограничивать контейнеры через capabilities и --user

Понимать принцип эшелонированной защиты

Главный вывод: Безопасность строится на нескольких слоях. Каждый слой ограничивает свои риски.
