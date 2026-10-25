pgbasebackup_dkr
================

Физический бэкап PostgreSQL через `pg_basebackup`, запущенный в Docker-контейнере.
На хост не устанавливается PostgreSQL-клиент: нужны только Docker CLI/daemon и
образ с `pg_basebackup`. Если образа нет, Docker скачает его при запуске.

Роль держит **одну локальную копию** бэкапа. Второй этап — duplicity, который
забирает её в хранилище. Старые копии не сохраняются: перед новым бэкапом текущий
каталог удаляется, новый создаётся на его месте.

Requirements
------------

* Docker Engine и CLI на хосте;
* запущенный контейнер с PostgreSQL (или внешняя БД и настроенная Docker-сеть);
* образ с `pg_basebackup`; при отсутствии Docker скачает его;
* пользователь PostgreSQL с правом репликации или суперпользователь;
* свободное место на файловой системе каталога бэкапов.

Role Variables
--------------

Обязательные:

| Переменная | Описание |
|------------|----------|
| `bkp_name` | префикс имени задачи |
| `pgdkr_pgpass` | пароль пользователя PostgreSQL (из vault/secret manager) |

Необязательные:

| Переменная | По умолчанию | Описание |
|------------|--------------|----------|
| `pgdkr_db_container` | пусто | имя/id контейнера PostgreSQL; пусто — автоопределение |
| `postgres_version` | пусто | общая инвентарная версия PostgreSQL, например `"16"`; используется для выбора образа/контейнера |
| `pgdkr_image` | пусто | явный образ для `pg_basebackup`; имеет приоритет |
| `pgdkr_network` | пусто | сеть backup-контейнера; пусто — `container:<найденный контейнер>` |
| `pgdkr_pghost` / `pgdkr_pgport` | `127.0.0.1` / `5432` | адрес PostgreSQL с точки зрения backup-контейнера |
| `pgdkr_pguser` | `postgres` | пользователь PostgreSQL |
| `pgdkr_pgdata` | `/var/lib/postgresql/data` | `PGDATA` внутри контейнера БД; для оценки размера |
| `pgdkr_basebackup_options` | `-c fast` | дополнительные опции `pg_basebackup` |
| `pgdkr_container_user` | `0:0` | пользователь backup-контейнера; root нужен для записи в root-каталог |
| `pgdkr_docker_bin` | `docker` | путь/имя docker CLI |
| `pgdkr_space_margin_percent` | `20` | запас свободного места сверх размера `PGDATA`, % |
| `pgdkr_min_free_kb` | `0` | резервный минимум, если размер не удалось оценить |
| `backup_dir` | `/home/backup` | корень каталогов роли |
| `node_exporter_textfile_dir` | не задана | каталог метрик; если пуст/отсутствует — метрика не пишется |
| `pgdkr_cron_minute` / `pgdkr_cron_hour` | `03` / `0` | расписание cron |

### Версия PostgreSQL и образ

`postgres_version` — общая переменная инвентаря; её же сможет использовать будущая
роль раскатки «коммунального» PostgreSQL. Образ выбирается так:

1. `pgdkr_image`, если задан;
2. иначе — образ найденного контейнера БД (`docker inspect ... Config.Image`);
3. иначе — `postgres:<postgres_version>`, если переменная задана;
4. иначе — определение версии PostgreSQL на хосте
   (`psql --version`, `pg_config --version`, `postgres --version`,
   `/usr/lib/postgresql/*/bin/postgres`) и образ `postgres:<major>`;
5. если версию определить не удалось — роль останавливается с требованием задать
   `postgres_version` в инвентаре или `pgdkr_image`.

Контейнер БД выбирается так:

* `pgdkr_db_container`, если задан;
* иначе среди запущенных контейнеров с образом PostgreSQL:
  * один — используется он;
  * несколько — выбирается совпадающий с `postgres_version`;
  * ни одного — если задан `pgdkr_network`, можно работать с внешней БД
    (`pgdkr_pghost`), иначе роль останавливается с требованием задать
    `pgdkr_db_container`.

Пути намеренно совпадают с `pgbasebackup_native`:

```
<duplicitybkp_path>/bin/<job>.sh
<duplicitybkp_path>/log/<job>.log
<duplicitybkp_path>/data/<job>
```

`pgdkr_jobname` по умолчанию — `<bkp_name>_pgbasebackup`, как у
`pgbasebackup_native`. Это позволяет заменить native-реализацию на Docker без
изменения путей в duplicity.

### Связь с duplicity

* SSH-режим duplicity по умолчанию забирает весь `duplicitybkp_data_path`, то есть
  каталог `data`, внутри которого лежит `<job>`;
* для Swift/S3 задайте `backup_target` на `duplicitybkp_data_path` или на
  конкретный `<job>`;
* `backup_dir`/`duplicitybkp_path` должны совпадать с настройками duplicity-роли.

Как работает
------------

1. Проверяет Docker CLI, находит контейнер БД и образ.
2. Оценивает размер `PGDATA` через `docker exec ... du -sk`; если не получилось —
   по текущему бэкапу, затем по `pgdkr_min_free_kb`. Сравнивает с `df -Pk`.
3. Создаёт временный env-файл `0600` с `PGPASSWORD`, чтобы пароль не попал в
   аргументы `docker run`/`ps`.
4. Удаляет текущий бэкап и создаёт новый на его месте:
   `docker run --rm --network ... --entrypoint pg_basebackup ...`.
5. При ошибке удаляет недоделанный каталог, временный env-файл и выходит с `1`;
   при успехе — `0`.
6. Выставляет `0700` на каталог, пишет метрику best-effort; лог — `0640`.

Важно: пока `pg_basebackup` работает, каталог бэкапа неполный. duplicity не должен
запускаться одновременно — разнесите расписание или используйте общую блокировку.

Example Playbook
----------------

```yaml
- hosts: dbservers
  become: true
  roles:
    - role: xoyteam.duplicitybkp.pgbasebackup_dkr
      vars:
        bkp_name: pg01
        backup_dir: /var/backups/pg
        postgres_version: "16"
        pgdkr_pgdata: /var/lib/postgresql/data
        pgdkr_pguser: postgres
        pgdkr_pgpass: change-me
        node_exporter_textfile_dir: /var/lib/node_exporter/textfile
```

`pgdkr_db_container` можно не задавать, если запущен ровно один контейнер
PostgreSQL. Если контейнеров несколько:

```yaml
        pgdkr_db_container: postgres16
```

Если PostgreSQL слушает не 5432 или доступен по имени в пользовательской
Docker-сети:

```yaml
        pgdkr_network: app_net
        pgdkr_pghost: postgres16
        pgdkr_pgport: 5432
```

License
-------

GPLv3

Author Information
------------------

xoy dot mail at gmail dot com
