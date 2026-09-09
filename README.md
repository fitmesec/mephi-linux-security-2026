# ДЗ №1. Изучение средств защиты ОС GNU/Linux

**Студент:** бухгалтерский шифр 373175
**ОС:** Fedora Workstation 44 (aarch64), виртуальная машина UTM
**Дата выполнения:** 09.09.2026

> ⚠️ Файлы `shadow` и `sudoers` содержат хэши паролей и настройки прав.
> Понимаю, что публикация `/etc/shadow` в реальной системе недопустима.
> Хэши принадлежат изолированной учебной ВМ, пароли нигде не переиспользуются.

---

## Раздел 1. Создание пользователя

| Что требовалось | Команда | Артефакт |
|---|---|---|
| Пользователь user1 с UID 1234 | `sudo useradd -u 1234 -m -s /bin/bash -G students user1` | `passwd`, `stat.out` |
| Группа students | `sudo groupadd students` | `group` |
| Смена пароля каждые 3 месяца | `sudo chage -M 90 -W 7 user1` | `shadow` |

Группа `students` задана как **дополнительная** (`-G`), чтобы членство было видно в файле `/etc/group` (`students:x:1001:user1`). Максимальный срок действия пароля — 90 дней, виден в 5-м поле `/etc/shadow` и в выводе `chage -l`.

## Раздел 2. Мониторинг файлов и процессов

| Что | Команда | Артефакт |
|---|---|---|
| Файлы с set-UID | `find / -xdev -type f -perm -4000 -exec ls -ld {} \;` | `setuid_files.out` |
| Процессы EUID=0, RUID обычный | `ps -eo pid,ruid,euid,ruser,euser,comm \| awk '$2>=1000 && $3==0'` | `setuid_procs.out` |

Использована форма `-perm -4000` (с дефисом): «среди битов есть set-UID». Процесс с повышенными привилегиями пойман на работающей команде `passwd` (RUID 1234 user1, EUID 0 root).

## Раздел 3. Механизм set-UID

Копия `cat` → `/home/user1/mycat`, владелец root, бит set-UID.

sudo install -o root -g root -m 0755 /usr/bin/cat /home/user1/mycat
sudo chmod u+s /home/user1/mycat


`chmod` выполняется **после** `install`, т.к. смена владельца сбрасывает бит set-UID.

Доказательство: `user1` не может прочитать `/etc/shadow` штатным `cat` (Permission denied), но читает через `mycat` — эффективный UID становится равным владельцу (root). Артефакт: `stat.out` (режим 4755).

## Раздел 4. Механизм привилегий (capabilities)

Копия `chown` → `/home/user1/mychown` с привилегией `cap_chown`.

sudo install -o root -g root -m 0755 /usr/bin/chown /home/user1/mychown
sudo setcap cap_chown=ep /home/user1/mychown


Суффикс `=ep`: `p` — привилегия в разрешённом наборе, `e` — сразу активна (chown не умеет сам управлять привилегиями). Доказательство: штатный `chown` от user1 → Operation not permitted; `mychown` меняет владельца на root. Артефакт: `getcap.out`.

**set-UID vs capabilities:** set-UID даёт программе всю личность root (40+ полномочий), capability — одно конкретное полномочие. При компрометации первого атакующий получает root, второго — только зафиксированное право.

## Раздел 5. Механизм sudo

Правило в `/etc/sudoers` (через `visudo`):

user1 ALL=(root) /usr/bin/timedatectl set-time *, /usr/bin/timedatectl set-ntp *, /usr/bin/date -s *, /usr/bin/hwclock --systohc


Пути абсолютные (иначе можно подменить команду через PATH). Пароль не отключён (`NOPASSWD` не задан) — подтверждение личности. Проверено: `user1` меняет системное время через sudo. Артефакт: `sudoers`.

## Артефакты

| Файл | Раздел |
|---|---|
| `mephi-screenshot.png` | 6 |
| `history.out` | все |
| `stat.out` | 1, 3, 4 |
| `getcap.out` | 4 |
| `passwd`, `shadow`, `group` | 1 |
| `sudoers` | 5 |
| `setuid_files.out` | 2 |
| `setuid_procs.out` | 2 |

Все настройки сохраняются в файлах на диске и переживают перезагрузку.
