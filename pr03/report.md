# ПР №3. Права доступа Linux и управление пользователями

## 1. Пользователи и группы

Созданные пользователи:
| Пользователь | Группа | Роль |
|--------------|--------|------|
| alice | developers | Администратор |
| bob | developers | Разработчик |
| carol | auditors | Аудитор |


**Поля /etc/passwd (alice):** alice:x:1001:1001::/home/alice:/bin/bash
- alice - логин
- x - пароль в /etc/shadow
- 1001 - UID
- 1001 - GID
- /home/alice - домашняя папка
- /bin/bash - оболочка

**/etc/shadow:** хранит хэши паролей, права 640 (только root)

## 2. Права chmod/chown


- chmod 750: bob не создал файл (нет w у группы)
- chmod 770: bob создал hello.py
- carol: нет доступа (не в группе)

## 3. ACL


**Команда:** setfacl -m u:carol:r-x /srv/project/code
**Плюс ACL:** точечная выдача прав без изменения группы

## 4. sudo-политики


| Пользователь | Команды |
|--------------|---------|
| alice | ALL (NOPASSWD) |
| bob | /usr/bin/apt* |
| carol | journalctl, cat /var/log/* |

## 5. PAM

**required** - обязательный модуль, при ошибке доступ запрещён

**Требования:** minlen=8

**Для длины 12:** добавить minlen=12 в /etc/pam.d/common-password

## Выводы

Научился управлять пользователями, правами chmod/chown, ACL, sudo-политиками. Понял принцип наименьших привилегий.
