pgbasebackup_native
===================

Физический бэкап PostgreSQL через `pg_basebackup`.

Роль держит **одну локальную копию** бэкапа. Второй этап — duplicity, который
забирает её в хранилище. Старые копии не сохраняются: перед новым бэкапом текущий
каталог удаляется, новый создаётся на его месте.

Requirements
------------

* клиент `pg_basebackup` версии не ниже сервера;
* пользователь PostgreSQL с правом репликации или суперпользователь;
* путь `PGDATA` на целевом хосте, читаемый root;
* свободное место на файловой системе каталога бэкапов.

Role Variables
--------------

Обязательные:

| Переменная | Описание |
|------------|----------|
| `bkp_name` | префикс имени задачи |
| `pgbinbkp_pgpass` | пароль пользователя PostgreSQL (из vault/secret manager) |
| `pgbinbkp_pgdata` | путь к живому `PGDATA`, например `/var/lib/postgresql/16/main` |

Необязательные:

| Переменная | По умолчанию | Описание |
|------------|--------------|----------|
| `pgbinbkp_pghost` | `127.0.0.1` | хост PostgreSQL |
| `pgbinbkp_pgport` | `5432` | порт |
| `pgbinbkp_pguser` | `postgres` | пользователь |
| `pgbinbkp_basebackup_options` | `-c fast` | дополнительные опции `pg_basebackup` |
| `pgbinbkp_space_margin_percent` | `20` | запас свободного места сверх размера `PGDATA`, % |
| `backup_dir` | `/home/backup` | корень каталогов роли |
| `node_exporter_textfile_dir` | не задана | каталог метрик; если пуст/отсутствует — метрика не пишется |
| `pgbinbkp_cron_minute` / `pgbinbkp_cron_hour` | `03` / `0` | расписание cron |

Пути: `backup_dir/bin/<job>.sh`, `backup_dir/log/<job>.log`,
`backup_dir/data/<job>`.

Как работает
------------

1. Проверяет, что `PGDATA` существует и является каталогом.
2. Считает размер `PGDATA` через `du -sk`, добавляет `pgbinbkp_space_margin_percent`
   и сравнивает с доступным местом (`df -Pk`). Место, занятое текущим бэкапом,
   учитывается как освобождаемое.
3. Удаляет текущий локальный бэкап и создаёт новый на его месте.
4. При ошибке `pg_basebackup` удаляет недоделанный каталог и завершается с кодом
   `1`; при успехе — `0`.
5. Выставляет `0700` и `umask 077`; лог — `0640`.
6. Пишет метрику `duplicity_backup{type="pg_basebackup_native"}` best-effort:
   ошибка метрики не отменяет успешный бэкап.

Важно: пока идёт `pg_basebackup`, каталог бэкапа неполный. duplicity не должен
запускаться одновременно с бэкапом — либо разносите расписание, либо используйте
общую блокировку.

Example Playbook
----------------

```yaml
- hosts: dbservers
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

Восстановление — штатное для `pg_basebackup`: остановить кластер, заменить
содержимое `PGDATA` содержимым каталога бэкапа, выставить права и запустить.

License
-------

GPLv3

Author Information
------------------

xoy dot mail at gmail dot com
