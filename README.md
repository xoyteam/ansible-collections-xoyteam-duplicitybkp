# xoyteam.duplicitybkp

Ansible-коллекция для резервного копирования:

* файлов и каталогов через [duplicity](https://duplicity.gitlab.io/) в Swift, S3 или
  на SSH/rsync-хранилище;
* баз MySQL (`mysqldump`) по расписанию;
* PostgreSQL физическим бэкапом через `pg_basebackup`;
* вспомогательные роли для запуска PostgreSQL в Docker.

Коллекция выросла из роли `duplicitybkp` (ранее лежала в Bitbucket), поэтому в ней
сочетаются несколько поколений задач: рабочая связка «duplicity + скрипты + cron»,
новая физическая бэкап-роль PostgreSQL и несколько ролей-заготовок.

> **Репозиторий публичный.** Реальных паролей, ключей, токенов, IP и имён хостов в
> нём быть не должно. Все реквизиты передаются переменными из потребляющего проекта
> (vault, secret manager, переменные окружения). В примерах ниже — только заглушки.

Версия коллекции: `0.0.1` (тегов и релизов нет). CI и тестов в репозитории нет.

---

## Содержание

1. [Состав коллекции](#состав-коллекции)
2. [Роль `duplicitybkp`](#роль-duplicitybkp)
3. [Роль `pgbasebackup_native`](#роль-pgbasebackup_native)
4. [Роли `pg_docker` и `dkr_pgbackup`](#роли-pg_docker-и-dkr_pgbackup)
5. [Роль `install_duplicity`](#роль-install_duplicity)
6. [Зависимости и установка](#зависимости-и-установка)
7. [Безопасность](#безопасность)
8. [Известные проблемы и подводные камни](#известные-проблемы-и-подводные-камни)
9. [История: что и зачем менялось](#история-что-и-зачем-менялось)
10. [Лицензия](#лицензия)

---

## Состав коллекции

```
galaxy.yml                  метаданные коллекции, версия 0.0.1
requirements.yml            исторические внешние зависимости (устарели)
requirements.txt            пины Python/Ansible для разработки (устарели)
plugins/                    пусто (только README-заглушка)
roles/
├── duplicitybkp/           основной «комбайн»: duplicity + mysql + swift/s3/ssh
├── pgbasebackup_native/    физический бэкап PostgreSQL через pg_basebackup
├── pg_docker/              запуск PostgreSQL в docker compose (заготовка)
├── dkr_pgbackup/           бэкап-сервис в Docker (заготовка, нерабочая)
└── install_duplicity/      пустая заготовка
```

| Роль | Назначение | Состояние |
|------|------------|-----------|
| `duplicitybkp` | Устанавливает duplicity и по переменной `bkptype` генерирует скрипты, конфиги, cron и logrotate для файловых бэкапов, MySQL, Swift, S3, SSH | Рабочая, но требует внешних переменных; см. ограничения |
| `pgbasebackup_native` | `pg_basebackup` + cron + logrotate + метрика для node_exporter | Рабочая, добавлена в мае 2025 |
| `pg_docker` | Поднимает контейнер PostgreSQL через `docker compose` | Заготовка, handler содержит ошибку |
| `dkr_pgbackup` | Заготовка compose-сервиса для бэкапа PostgreSQL | Фактически нерабочая: шаблон скопирован от postgres-exporter |
| `install_duplicity` | Должна была ставить duplicity | Пустая, не используется |

---

## Роль `duplicitybkp`

Это роль-диспетчер. Всё поведение определяется одной переменной `bkptype`: роль
подключает соответствующий набор задач и шаблонов.

### Как работает диспетчер

`tasks/main.yml`:

1. Для режимов `duplicity`, `duplicitybkp_ssh`, `ssh_key`, а также для любого
   `bkptype`, содержащего подстроку `swift` или `s3`, импортирует `duplicity.yml`:
   ставит пакеты, создаёт каталоги `bin`, `etc`, `data`, `log`.
2. Затем по точному значению `bkptype` подключает один набор задач.

Важно: проверки вида `'s3' in bkptype` — это проверка подстроки, а не равенство.
Режимы `mysql`, `mysql_fullcfg`, `mysqlclient*` duplicity **не** устанавливают: они
только готовят дампы на локальном диске, выгрузку должен делать кто-то ещё.

### Режимы `bkptype`

В колонке «Ключевые переменные» перечислено то, что обычно задаёт потребитель;
остальные значения есть в `defaults` (см. ниже).

| `bkptype` | Что делает | Ключевые переменные |
|-----------|------------|---------------------|
| `duplicity` | Только устанавливает duplicity и создаёт каталоги | — |
| `ssh_key` | Генерирует ed25519-ключ в `duplicitybkp_cfg_path` и добавляет публичную часть в `authorized_keys` на хосте-хранилище | `duplicitybkp_ssh_store_user`, `duplicitybkp_ssh_store_hostname` |
| `duplicitybkp_ssh` | Скрипт `duplicity` поверх `rsync://user@host/path`, cron, logrotate | `duplicitybkp_ssh_store_user`, `duplicitybkp_ssh_store_host`; цель — `duplicitybkp_ssh_backup_target` |
| `swift` | Скрипт `dupliswift.sh` (упрощённый, реквизиты зашиты в шаблон), exclude-лист, logrotate, cron | `backup_name`, `backup_target`, `backup_excludelist`, `backup_extraopts`, `backup_swiftuser`, `backup_swiftpassword`, `backup_swiftbucket` |
| `swift_fullcfg` | То же, но реквизиты Swift берутся из переменных, пишется метрика для node_exporter | То же + `duplicitybkp_swift_*` |
| `duplicitybkp_s3` | Скрипт выгрузки в S3, cron, метрика | `backup_name`, `backup_target`, `backup_excludelist`, `backup_extraopts`, `duplicitybkp_s3_bucket`, `duplicitybkp_s3_accesskey`, `duplicitybkp_s3_secretkey`, `duplicitybkp_s3_endpoint_url`, `duplicitybkp_s3_region_name` |
| `mysql` | Старый скрипт `/usr/local/sbin/mysqlbkp`, конфиг `/etc/<bkpprefix>mysqlbkp/mysql`, cron, logrotate | `bkpprefix`, `mysqlbkp_host`, `mysqlbkp_pass` |
| `mysql_fullcfg` | Скрипт в `duplicitybkp_bin_path`, конфиг со всеми реквизитами в строке, cron, logrotate, метрика | `mysqlbkp_host`, `mysqlbkp_pass`, `node_exporter_textfile_dir` |
| `mysqlclient` | Добавляет строку `db путь` в общий конфиг MySQL-бэкапа | `bkpprefix`, `bkp_path`, `mysql_db` |
| `mysqlclient_fullcfg` | Добавляет строку `db путь host port user pass` в конфиг | `mysqlbkp_db`, `mysqlbkp_data_path`, `mysqlbkp_host`, `mysqlbkp_port`, `mysqlbkp_pass` |

Типовой порядок для MySQL со всеми реквизитами в конфиге:

1. `bkptype: mysql_fullcfg` — ставит скрипт и cron;
2. `bkptype: mysqlclient_fullcfg` — добавляет по строке на каждую базу.

Для SSH-бэкапа keyless-схема требует двух прогонов: сначала `ssh_key` (создать ключ
и положить публичную часть на хранилище), затем `duplicitybkp_ssh` (создать скрипт
и cron).

### Переменные

#### Общие

| Переменная | По умолчанию | Смысл |
|------------|--------------|-------|
| `bkptype` | не задана | Режим работы роли (см. таблицу выше) |
| `bkp_name` | `test` | Префикс имён задач, ключей, метрик |
| `backup_user` | `backup` | Группа-владелец каталогов бэкапов; пользователь не создаётся |
| `backup_dir` | `duplicitybkp_path`, то есть `/usr/local` | Корень для скриптов swift/s3; подкаталог `backup_name` роль не создаёт |

#### Установка duplicity

| Переменная | По умолчанию | Смысл |
|------------|--------------|-------|
| `duplicitybkp_bin_type` | `deb` | `deb` — пакет из apt, `pip` — установка в отдельный Python |
| `duplicitybkp_path` | `/usr/local` | Корень каталогов `bin`, `etc`, `data`, `log` |
| `duplicitybkp_duplicity_path` | `/usr/bin/duplicity` | Путь к бинарнику duplicity |
| `duplicitybkp_pip_path` | `/srv/python3.10/bin/pip3.10` | pip для режима `pip` |
| `duplicitybkp_packages_list_pip` | duplicity 2.2.1, swiftclient, keystoneclient, fasteners | Список пакетов для режима `pip` |

При `bin_type: deb` роль ставит `gnupg2`, `librsync-dev`, `duplicity` и дополнительно
`python-swiftclient`/`python-boto3` для swift/s3. Если пакета `python-pip` нет
(актуально для Debian 12+), срабатывает `rescue`-блок с `python3-pip` и
`python3-boto3`.

#### SSH-хранилище

| Переменная | По умолчанию | Смысл |
|------------|--------------|-------|
| `duplicitybkp_ssh_store_proto` | `rsync` | Схема DSN |
| `duplicitybkp_ssh_store_user` | не задана | Пользователь на хранилище |
| `duplicitybkp_ssh_store_host` | не задана | Хост в DSN (может быть IP) |
| `duplicitybkp_ssh_store_hostname` | `duplicitybkp_ssh_store_host` | Имя для `delegate_to` (должно резолвиться с control-node) |
| `duplicitybkp_ssh_store_path` | имя задачи | Путь на хранилище |
| `duplicitybkp_ssh_backup_target` | `duplicitybkp_data_path` | Что копируем |
| `duplicitybkp_ssh_extraopts` | `--rsync-options=...IdentityFile...` | Доп. опции duplicity для SSH |
| `duplicitybkp_ssh_exclude_list` | не задана | Список исключений (см. известные проблемы) |
| `duplicitybkp_volsize` | `500` | Размер тома duplicity, МБ |
| `duplicitybkp_removeolder` | `31D` | Удалять тома старше |
| `duplicitybkp_fullcopies` | `3` | Сколько полных копий хранить |
| `duplicitybkp_extraopts` | пусто | Доп. опции duplicity |

#### Swift и S3: историческое разделение имён

Скрипты swift/s3 — самые старые в коллекции и читают **не** `duplicitybkp_*`, а
собственный набор `backup_*`, которого нет в `defaults/main.yml`. Эти переменные
обязаны быть заданы потребителем:

`backup_dir`, `backup_name`, `backup_target`, `backup_excludelist`,
`backup_extraopts`, `backup_volsize`, `backup_fullevry`, `backup_removeolder`,
`backup_fullcopies`.

Для Swift дополнительно: `backup_swiftuser`, `backup_swiftpassword`,
`backup_swiftbucket`; в режиме `swift_fullcfg` ещё `duplicitybkp_swift_authurl`,
`duplicitybkp_swift_auth_version`, `duplicitybkp_swift_tenantname`,
`duplicitybkp_swift_userdomain`, `duplicitybkp_swift_projectdomain`.

Для S3 дополнительно: `duplicitybkp_s3_bucket`, `duplicitybkp_s3_accesskey`,
`duplicitybkp_s3_secretkey`, `duplicitybkp_s3_endpoint_url`,
`duplicitybkp_s3_region_name`.

> Имя `backup_fullevry` — не опечатка в этом README, а именно так называется
> переменная в шаблонах (от `full every`).

#### MySQL

| Переменная | По умолчанию | Смысл |
|------------|--------------|-------|
| `mysqlbkp_user` | `bkp` | Пользователь MySQL |
| `mysqlbkp_host` / `mysqlbkp_port` | не заданы / `3306` | Сервер MySQL |
| `mysqlbkp_dir_group` | `adm` | Группа-владелец каталогов (0750): обзор без sudo, чтение файлов — только root |
| `mysqlbkp_credentials_dir` | `{{ playbook_dir }}/credentials-crypt` | Каталог с секретами (git-crypt) в потребляющем проекте |
| `mysqlbkp_pass_file` | `<dir>/<host>/mysql_bkp_password` | Файл с паролем; можно переопределить |
| `mysqlbkp_pass` | читается из `mysqlbkp_pass_file` | Пароль; пустой/отсутствующий файл → ошибка. Явно заданное значение имеет приоритет |
| `mysqlbkp_compress` | `false` | Сжатие gzip в режиме `mysql_fullcfg` |
| `bkpprefix` | не задана | Префикс каталога `/etc/<bkpprefix>mysqlbkp` в старом режиме |
| `mysql_db`, `bkp_path` | не заданы | Строка для `mysqlclient` |
| `mysqlbkp_db`, `mysqlbkp_data_path` | не заданы | Строка для `mysqlclient_fullcfg` |
| `node_exporter_textfile_dir` | не задана | Каталог textfile-метрик; в шаблоне используется без `default` |

#### Где лежит пароль MySQL

Пароль не генерируется ролью. По умолчанию он читается на control-node из файла
`credentials-crypt/<mysqlbkp_host>/mysql_bkp_password` рядом с playbook
(`{{ playbook_dir }}/credentials-crypt/...`):

```
<project>/
├── site.yml
└── credentials-crypt/            # расшифровывается git-crypt в потребляющем проекте
    └── db.example.com/
        └── mysql_bkp_password    # одна строка, 0600, не пустая
```

Файл должен быть расшифрован (`git-crypt unlock`) до запуска. Если файла нет, он
пуст или недоступен для чтения — роль останавливается с ошибкой. Явно заданный
`mysqlbkp_pass` имеет приоритет, и файл в этом случае не читается. Пути можно
переопределить через `mysqlbkp_credentials_dir` и `mysqlbkp_pass_file`.

> В самом репозитории коллекции каталога `credentials-crypt/` нет и быть не должно:
> это git-crypt-каталог потребляющего проекта.

В режиме `mysql_fullcfg` пароль записывается в конфиг как `b64:<base64>` — так
пробелы и спецсимволы не ломают разбор строки. Старые строки без префикса `b64:`
читаются как обычный пароль (обратная совместимость). В старом режиме `mysql`
пароль попадает в скрипт через shell-quoting, поэтому спецсимволы также безопасны.

#### Права и запуск

Каталоги, которыми управляет роль (`/etc/<bkpprefix>mysqlbkp`,
`mysqlbkp_config_path`, `mysqlbkp_data_path`, `mysqlbkp_log_path`), — `0750`
`root:<mysqlbkp_dir_group>` (по умолчанию `adm`). Члены группы могут войти в
каталог и увидеть список файлов, но не читать их. Файлы: конфиги, дампы и логи —
`0600 root:root`, скрипты — `0700 root:root`. Дампы создаются с `umask 077` и
явным `chmod 600`, логи — с `umask 077` в cron; logrotate создаёт новые логи
`0600`. Чтение файлов — только root/sudo. Исключение — файлы метрик в
`node_exporter_textfile_dir`: они создаются `0644`, потому что их читает
node_exporter; секретов в них нет.

Cron-задания без параметра `user` выполняются от root (playbook с
`become: true`), поэтому root читает `0600` и запускает `0700`. Запуск роли без
`become` не сможет писать в `/usr/local` и `/etc` — это ограничение привилегий,
а не прав созданных файлов.

`mysqlbkp_bin_path` по умолчанию `/usr/local/bin` — общий системный каталог; роль
только проверяет его существование и не меняет группу/права. `mysqlbkp_config_path`
(`/usr/local/etc`) и `mysqlbkp_log_path` (`/usr/local/log`) по умолчанию тоже
общие: если на хосте это важно, задайте отдельные каталоги (например, через
`duplicitybkp_path`) или выберите `mysqlbkp_dir_group` осознанно.


### Примеры

Все значения ниже — заглушки.

#### Файловый бэкап по SSH/rsync

```yaml
# 1. Создаём ключ и разрешаем его на хосте-хранилище
- name: Prepare SSH key for backup
  hosts: fileservers
  become: true
  roles:
    - role: xoyteam.duplicitybkp.duplicitybkp
      vars:
        bkptype: ssh_key
        bkp_name: files
        duplicitybkp_path: /var/backups/duplicity
        duplicitybkp_ssh_store_user: backup
        duplicitybkp_ssh_store_hostname: backup.example.com

# 2. Ставим duplicity, создаём скрипт и cron
- name: Configure SSH backup
  hosts: fileservers
  become: true
  roles:
    - role: xoyteam.duplicitybkp.duplicitybkp
      vars:
        bkptype: duplicitybkp_ssh
        bkp_name: files
        duplicitybkp_path: /var/backups/duplicity
        duplicitybkp_ssh_store_user: backup
        duplicitybkp_ssh_store_host: 192.0.2.10
        duplicitybkp_ssh_store_hostname: backup.example.com
        duplicitybkp_ssh_backup_target: /srv/data
```

#### Swift

```yaml
- name: Files to Swift
  hosts: fileservers
  become: true
  tasks:
    - name: Каталог назначения (роль его не создаёт)
      ansible.builtin.file:
        path: /var/backups/duplicity/files
        state: directory
        mode: "0700"
  roles:
    - role: xoyteam.duplicitybkp.duplicitybkp
      vars:
        bkptype: swift_fullcfg
        backup_dir: /var/backups/duplicity
        backup_name: files
        backup_target: /srv/data
        backup_excludelist: ""
        backup_extraopts: ""
        backup_swiftuser: change-me
        backup_swiftpassword: change-me
        backup_swiftbucket: change-me
        duplicitybkp_swift_authurl: https://api.example.com/v2.0
        duplicitybkp_swift_auth_version: 2
        duplicitybkp_swift_tenantname: change-me
        duplicitybkp_swift_userdomain: ""
        duplicitybkp_swift_projectdomain: ""
```

#### S3

```yaml
- name: Files to S3
  hosts: fileservers
  become: true
  tasks:
    - name: Каталог назначения
      ansible.builtin.file:
        path: /var/backups/duplicity/files
        state: directory
        mode: "0700"
  roles:
    - role: xoyteam.duplicitybkp.duplicitybkp
      vars:
        bkptype: duplicitybkp_s3
        backup_dir: /var/backups/duplicity
        backup_name: files
        backup_target: /srv/data
        backup_excludelist: ""
        backup_extraopts: ""
        duplicitybkp_s3_bucket: change-me
        duplicitybkp_s3_accesskey: change-me
        duplicitybkp_s3_secretkey: change-me
        # обе переменные обязательны даже пустыми: на них ссылается defaults
        duplicitybkp_s3_endpoint_url: ""
        duplicitybkp_s3_region_name: ""
```

#### MySQL: скрипт + базы в конфиге

```yaml
- name: MySQL dumps
  hosts: dbservers
  become: true
  tasks:
    - name: 1. Скрипт и cron
      ansible.builtin.include_role:
        name: xoyteam.duplicitybkp.duplicitybkp
      vars:
        bkptype: mysql_fullcfg
        bkp_name: mysql01
        mysqlbkp_host: 127.0.0.1
        mysqlbkp_port: 3306
        mysqlbkp_user: bkp
        mysqlbkp_data_path: /var/backups/duplicity/data/mysql
        mysqlbkp_compress: true
        node_exporter_textfile_dir: /var/lib/node_exporter/textfile

    - name: 2. Добавить базу в конфиг
      ansible.builtin.include_role:
        name: xoyteam.duplicitybkp.duplicitybkp
      vars:
        bkptype: mysqlclient_fullcfg
        mysqlbkp_db: app
        mysqlbkp_data_path: /var/backups/duplicity/data/mysql
        mysqlbkp_host: 127.0.0.1
        mysqlbkp_port: 3306
        mysqlbkp_user: bkp
        # можно не задавать: пароль будет прочитан из
        # credentials-crypt/<mysqlbkp_host>/mysql_bkp_password
        mysqlbkp_pass: change-me
```

Дампы складываются в `mysqlbkp_data_path`, cron запускает скрипт ежедневно в 00:03,
logrotate — 7 дней. Выгрузку дампов в хранилище роль не делает.

---

## Роль `pgbasebackup_native`

Физический бэкап PostgreSQL через `pg_basebackup`. Появилась в мае 2025 как замена
схеме «копировать каталог PGDATA через duplicity».

Что делает роль:

1. Проверяет, что `pgbinbkp_pgdata` задан и является каталогом.
2. Создаёт каталог данных `pgbinbkp_data_path` (0700) и файл лога
   `pgbinbkp_log_name` (0640, `root:root`).
3. Рендерит скрипт `script.sh.j2` в `pgbinbkp_bin_name` (0700).
4. Ставит cron-задание (по умолчанию 03 минуты 00 часов) с дописыванием в лог.
5. Настраивает logrotate на 7 дней.

Скрипт проверяет свободное место по размеру `PGDATA` + запас, удаляет текущий
локальный бэкап, создаёт новый на его месте и оставляет только его. При ошибке
недоделанный каталог удаляется, код возврата — `1`; при успехе — `0`. Старые
копии не сохраняются: локальная копия нужна только до выгрузки duplicity.
Метрика `duplicity_backup{type="pg_basebackup_native",project="..."}` пишется
best-effort.

> Пока идёт `pg_basebackup`, каталог данных неполный. duplicity не должен
> запускаться одновременно: разнесите расписание или используйте общую
> блокировку.

### Переменные

| Переменная | По умолчанию | Обязательна |
|------------|--------------|-------------|
| `bkp_name` | нет | Да: от неё строится имя задачи |
| `backup_dir` | `/home/backup` | Нет |
| `pgbinbkp_pgdata` | нет (пустая) | Да: путь к `PGDATA` на хосте |
| `pgbinbkp_pgpass` | нет (пустая) | Да: пароль пользователя PostgreSQL |
| `pgbinbkp_pghost` | `127.0.0.1` | Нет |
| `pgbinbkp_pgport` | `5432` | Нет |
| `pgbinbkp_pguser` | `postgres` | Нет |
| `pgbinbkp_basebackup_options` | `-c fast` | Нет |
| `pgbinbkp_space_margin_percent` | `20` | Нет |
| `pgbinbkp_cron_minute` / `pgbinbkp_cron_hour` | `03` / `0` | Нет |
| `node_exporter_textfile_dir` | нет | Нет: если пусто/нет каталога, метрика не пишется |

Пути: `backup_dir/bin/<job>.sh`, `backup_dir/log/<job>.log`,
`backup_dir/data/<job>`.

### Пример

```yaml
- name: PostgreSQL physical backup
  hosts: dbservers
  become: true
  roles:
    - role: xoyteam.duplicitybkp.pgbasebackup_native
      vars:
        bkp_name: pg01
        backup_dir: /var/backups/pg
        pgbinbkp_pgdata: /var/lib/postgresql/16/main
        pgbinbkp_pghost: 127.0.0.1
        pgbinbkp_pgport: 5432
        pgbinbkp_pguser: postgres
        pgbinbkp_pgpass: change-me
        node_exporter_textfile_dir: /var/lib/node_exporter/textfile
```

Требования к хосту: установленный клиент `pg_basebackup` (версия не ниже сервера),
пользователь с правом репликации или суперпользователь, доступ к серверу по сети
или через локальный сокет, читаемый root путь `pgbinbkp_pgdata`. Восстановление —
штатное для `pg_basebackup`: остановить кластер, заменить содержимое `PGDATA`
содержимым каталога бэкапа, выставить права и запустить.

---

## Роли `pg_docker` и `dkr_pgbackup`

Обе роли — заготовки из шаблона cookiecutter, а не готовый инструмент.

`pg_docker` рендерит compose-файл с `postgres:<tag>` (по умолчанию 13.20), порт
`5433:5432`, том `./data`, tmpfs. Роль создаёт каталоги и compose-файл, но не
запускает контейнер. Handler использует устаревший модуль `docker_compose` и
обращается к незарегистрированной переменной `pgdocker_docker_compose`.

`dkr_pgbackup` ещё дальше от рабочего состояния: в compose-файле образ PostgreSQL,
но environment и healthcheck (`localhost:9187/metrics`) скопированы от
postgres-exporter; handler ссылается на несуществующую
`dkr_pgbackup_exporter_install_path`; шаблон использует `postgres_login_*`, а
defaults объявляют `dkr_pgbackup_login_*`.

Обе роли используют модуль `docker_compose` (v1). Для новых Docker нужен
`community.docker.docker_compose_v2`. Роли не рекомендуется применять без
переработки.

---

## Роль `install_duplicity`

Пустая заготовка: `tasks/main.yml` содержит только комментарии, `defaults` —
`python_bin_path`. Нигде не подключается. Установка duplicity фактически живёт в
роли `duplicitybkp` (`tasks/duplicity.yml`).

---

## Зависимости и установка

### Что реально нужно

| Зависимость | Зачем |
|-------------|-------|
| `community.crypto` | `openssh_keypair` в режиме `ssh_key` |
| `ansible.posix` | `authorized_key` в режиме `ssh_key` |
| `community.docker` | только для `pg_docker` / `dkr_pgbackup` |
| `pg_basebackup` на хосте | роль `pgbasebackup_native` |
| `duplicity` | устанавливается ролью, но можно поставить заранее |

`requirements.yml` в репозитории этим требованиям **не соответствует**: в нём
перечислены `community.grafana`, `beerfranz.wtu_apps`, `xoyteam.docker_compose_apps`
(нигде не используются) и старинная роль `duplicitybkp` из Bitbucket, а
`community.crypto` и `ansible.posix` отсутствуют. Перед использованием обновите
файл под свой проект.

### Установка

```bash
# из исходников
ansible-galaxy collection build
ansible-galaxy collection install xoyteam-duplicitybkp-0.0.1.tar.gz

# или просто положить каталог в коллекции проекта
# <project>/collections/ansible_collections/xoyteam/duplicitybkp
```

Вызов роли в playbook:

```yaml
- role: xoyteam.duplicitybkp.duplicitybkp
```

### Проверка

```bash
ansible-playbook --syntax-check site.yml
```

Полноценный прогон — только по явной команде и с `--limit <host>`: роль создаёт
cron-задания и перезаписывает конфиги.

---

## Безопасность

* Роль не генерирует и не хранит пароль MySQL. По умолчанию он читается из
  `credentials-crypt/<mysqlbkp_host>/mysql_bkp_password` — это git-crypt-каталог
  потребляющего проекта, а не этой коллекции. Файл должен быть расшифрован до
  запуска; пустой или отсутствующий секрет приводит к ошибке.
* Скрипты MySQL не трассируют секреты: `set -x` выключается вокруг присваивания
  реквизитов и вызова `mysqldump`, при этом подробный лог сохраняется. Пароль
  по-прежнему передаётся через `-p"$PASSWD"` и виден в `ps` — это отдельный
  известный дефект.
* В `pgbasebackup_native` глобальный `set -x` убран; `PGPASSWORD` передаётся через
  окружение и не попадает в лог. Переход на `PGPASSFILE` — отдельный шаг: пока
  пароль виден в `/proc/<pid>/environ` для root.
* Все скрипты и конфиги содержат реквизиты в открытом виде или в base64 (это не
  шифрование): пароль MySQL, `PGPASSWORD`, Swift-пароль, AWS-ключи. Каталоги
  бэкапов — `0750 root:<mysqlbkp_dir_group>` (по умолчанию `adm`), файлы —
  `0600`/`0700`: администраторы из группы видят список файлов, но читают их
  только через sudo.
* Все вызовы duplicity идут с `--no-encryption`: данные лежат в хранилище без
  клиентского шифрования. Рассчитывайте на шифрование на стороне хранилища либо
  добавляйте GPG-опции через `duplicitybkp_extraopts` / `backup_extraopts`.
* Пароль MySQL передаётся в командной строке (`-p"$PASSWD"`) и виден в `ps`.
  `PGPASSWORD` в `pg_basebackup` передаётся через окружение и в `ps` не виден, но
  доступен в `/proc/<pid>/environ` для root.
* Публичный репозиторий: не коммитьте inventory, `group_vars`, ключи, дампы,
  реальные IP и учётные записи. Все примеры — только заглушки `change-me`.

---

## Известные проблемы и подводные камни

Это не «пожелания», а реальные дефекты текущего кода. Учитывайте их до запуска.

**Переменные и имена**

1. Пароль MySQL передаётся в `mysqldump` через `-p"$PASSWD"` и виден в `ps`
   (в логах он замаскирован). Убрать — отдельным шагом через
   `--defaults-extra-file`.
2. Скрипты swift/s3 читают `backup_*`, а defaults описывают `duplicitybkp_*`.
   Общих значений у них нет, `backup_name`/`backup_target`/`backup_excludelist`
   и т. п. нужно задавать отдельно.
3. Опечатка в задаче: exclude-файл создаётся при `duplicitybkp_ssh_execlude_list`
   (лишняя `e`), а шаблон проверяет правильное `duplicitybkp_ssh_exclude_list`.
   Чтобы исключения заработали, приходится задавать обе.
4. В `duplicitybkp_ssh.sh.j2` используется `${EXCLUDE_FILE}`, но переменная нигде
   не определяется. При включённых исключениях duplicity получит пустой путь.
5. `duplicitybkp_s3_endpoint_url` и `duplicitybkp_s3_region_name` обязательны даже
   пустыми: defaults вычисляют от них `duplicitybkp_s3_opts` через `|length`.
6. `node_exporter_textfile_dir` не объявлена в defaults, но без неё не
   отрендерятся шаблоны MySQL/pg_basebackup. Каталог роль тоже не создаёт.

**Каталоги и права**

7. Для swift/s3 роль пишет в `{{ backup_dir }}/{{ backup_name }}`, но этот каталог
   не создаёт. На чистом хосте задача `template` упадёт.
8. `backup_dir` в `duplicitybkp` по умолчанию `/usr/local`, а в
   `pgbasebackup_native` — `/home/backup`. Единого значения нет.
9. Роль создаёт группу `backup_user`, но не пользователя; cron-задания работают
   от root. Права созданных файлов рассчитаны именно на root (см. «Права и
   запуск»).

**Обработка ошибок**

10. `mysqlbkp.sh.j2`: `rc=$?` берётся после `du`, а не после `mysqldump`, поэтому
    ошибка дампа маскируется. Пайплайн с `gzip` без `pipefail`.
11. MySQL-скрипты (`mysqlbkp.sh.j2`, `mysqlbkp_fullcfg.sh.j2`) завершаются
    `exit 0` при любом исходе — cron не увидит сбой. В `pgbasebackup_native`
    коды возврата уже исправлены.
12. `pgbasebackup_native` не хранит предыдущую копию: при ошибке `pg_basebackup`
    недоделанный каталог удаляется, локального бэкапа не остаётся. Так и задумано —
    следите, чтобы duplicity успевал забрать предыдущую копию до следующего
    запуска.
13. `mysqlbkp_compress: false` рендерится в скрипт как строка `False`, а скрипт
    сравнивает с `true`. В итоге сжатие не включается, пока не задать строку
    `"true"`.

**Прочее**

14. `pg_docker`: handler вызывает `docker_compose` (v1) и переменную
    `pgdocker_docker_compose`, которая нигде не регистрируется.
15. `dkr_pgbackup`: шаблон и handler — от postgres-exporter, роль нерабочая.
16. Мёртвые файлы: `tasks/ssh_duplicity.yml` и `tasks/duplicity_install_pip.yml`
    нигде не подключаются.
17. `install_duplicity` пуста.
18. `galaxy.yml` ссылается на `github.com/xoyteam/collection-general` во всех
    полях (repository, documentation, homepage, issues).
19. `requirements.txt` пинит `ansible==4.8.0`, `ansible-base==2.10.15`,
    `ansible-core==2.11.9` — набор несовместимых между собой версий; для
    современного Ansible не используйте его напрямую.
20. Тестов, molecule-сценариев и CI нет.

---

## История: что и зачем менялось

В репозитории 6 коммитов (май–июнь 2025), ветка одна — `main`, тегов нет.

| Коммит | Что сделано | Зачем |
|--------|-------------|-------|
| `193324d` | Первичный импорт коллекции: 75 файлов, 3306 строк | Перенос роли `duplicitybkp` из Bitbucket (`requirements.yml` до сих пор ссылается на `git@bitbucket.org:xoy/duplicitybkp.git`, тег `v1.0`) и добавление ролей `pg_docker`, `dkr_pgbackup`, `install_duplicity`. Роль `pgbasebackup_duplicitybkp` была **точной копией** `duplicitybkp`. README был скопирован от `docker_compose_apps`, а в `galaxy.yml` осталась версия `1.0` |
| `46f8b42` | README: заголовок `xoyteam.docker_compose_apps` заменён на `xoyteam.duplicitybkp`, добавлено описание | Убрать следы копипаста из другого проекта |
| `a7cef11` | `galaxy.yml`: версия `1.0` → `0.0.1` | Galaxy требует SemVer `x.y.z`; `1.0` — невалидная версия |
| `e81b70f` | Ключевой коммит: `pgbasebackup_duplicitybkp` переименована в `pgbasebackup_native`, из неё удалены 900 строк копии duplicity-роли; добавлены собственные defaults, tasks и шаблон `pg_basebackup` (31 файл, −899/+93). Заодно в `duplicitybkp/tasks/ssh_key.yml`: `delegate_to` переключён с `duplicitybkp_ssh_store_host` на `duplicitybkp_ssh_store_hostname` | Вместо копирования PGDATA файловым бэкапом через duplicity появился полноценный физический бэкап `pg_basebackup`. Дублирующая роль удалена, чтобы не поддерживать две одинаковые копии. Разделение `store_host` (адрес в rsync-DSN, может быть IP) и `store_hostname` (имя для `delegate_to`, должно резолвиться с control-node) убирает ошибку делегирования при разных значениях |
| `abc08e3` | В `pgbasebackup_native/tasks/main.yml` цикл создания каталогов получил `state` для каждого элемента; добавлено создание файла лога с правами `0640`, `root:root` | Раньше лог создавался только перенаправлением cron с umask по умолчанию (0644) и мог не совпадать с `create 640 root root` в logrotate. Фикс делает права и владельца лога предсказуемыми |
| `421226d` | Из `pgbasebackup_native/templates/script.sh.j2` удалена строка `done < $BkpListFile` | Строка осталась от скопированного MySQL-скрипта с циклом `while read`. В скрипте без цикла `done` даёт синтаксическую ошибку bash — `pg_basebackup` вообще не запускался. Удаление делает скрипт исполняемым и валидным |

Итог эволюции: из одной роли `duplicitybkp` (файловые бэкапы, MySQL, Swift/S3/SSH)
выделилась отдельная роль для PostgreSQL, а «мусорные» копии и нерабочие заготовки
остались в репозитории как историческое наследие. Новые изменения стоит направлять
на исправление пунктов из раздела «Известные проблемы», не ломая существующие
имена переменных: коллекция публичная и уже используется в других проектах.

---

## Лицензия

GPL-3.0-or-later. См. [LICENSE](LICENSE).
