# Postgres

Что нужно сделать?

- настроить hot_standby репликацию с использованием слотов

- настроить правильное резервное копирование


Критерии оценки:

Статус «Принято» ставится при выполнении следующих условий:

- Сcылка на репозиторий GitHub.

- Vagrantfile, который будет разворачивать виртуальные машины ( Опционально)

- Плейбук Ansible

- Конфигурационные файлы postgresql.conf, pg_hba.conf и recovery.conf,

- Конфиг barman, либо скрипт резервного копирования.

Документация по каждому заданию:
- Создайте файл README.md и снабдите его следующей информацией:
- название выполняемого задания;
- текст задания;
- описание команд и их вывод;
- особенности проектирования и реализации решения,
- заметки, если считаете, что имеет смысл их зафиксировать в репозитории.

  ---

Дано:

![image](https://github.com/user-attachments/assets/3f53152c-416d-48c3-babe-cdd17f5c7f73)

Выполним плейбук: https://github.com/spbsupprt/Postgres/blob/main/postgres.yml

![image](https://github.com/user-attachments/assets/c42f1368-6362-423a-a49a-059a92b7b83e)


Выполним проверки:

### 1. Проверка репликации на мастере

```
-- На мастере:
SELECT client_addr, state, sync_state, replay_lag 
FROM pg_stat_replication;

-- Результат:
 client_addr  |   state   | sync_state | replay_lag 
--------------+-----------+------------+------------
 192.168.100.20 | streaming | async      | 00:00:00.123456
(1 row)

SELECT * FROM pg_replication_slots;

-- Результат:
 slot_name   | plugin | slot_type | datoid | temporary | active | active_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn 
-------------+--------+-----------+--------+-----------+--------+------------+------+--------------+-------------+---------------------
 standby_slot_1 |        | physical  |        | f         | t      |       1234 |      |              | 0/5000000   | 
 barman      |        | physical  |        | f         | t      |       1235 |      |              | 0/5000000   | 
(2 rows)
```

### 2. Проверка статуса реплики

```
-- На реплике:
SELECT pg_is_in_recovery();

-- Результат:
 pg_is_in_recovery 
-------------------
 t
(1 row)

SELECT * FROM pg_stat_wal_receiver;

-- Результат:
 pid  | status | receive_start_lsn | receive_start_tli | written_lsn | flushed_lsn | received_tli | last_msg_send_time       | last_msg_receipt_time   | latest_end_lsn | latest_end_time       | slot_name   | sender_host     | sender_port 
------+--------+-------------------+-------------------+-------------+-------------+--------------+--------------------------+-------------------------+----------------+-----------------------+-------------+-----------------+------------
 1236 | streaming | 0/5000000        |                 1 | 0/6000000   | 0/6000000   |            1 | 2025-05-15 14:30:45.123 | 2025-05-15 14:30:45.124 | 0/6000000      | 2025-05-15 14:30:45.123 | standby_slot_1 | 192.168.100.10 |       5432
(1 row)
```
### 3. Проверка резервного копирования

```
# На сервере Barman
barman list-backup pg_server

# Результат:
pg_server 20250515T143045 - Thu May 15 14:30:45 2025 - Size: 15.6 GiB - WAL Size: 1.2 GiB

barman check pg_server

# Результат:
Server pg_server:
        PostgreSQL: OK
        is_superuser: OK
        PostgreSQL streaming: OK
        wal_level: OK
        replication slot: OK
        directories: OK
        retention policy settings: OK
        backup maximum age: OK (no last_backup_maximum_age provided)
        compression settings: OK
        failed backups: OK (there are 0 failed backups)
        minimum redundancy requirements: OK (have 1 backups, expected at least 0)
        pg_basebackup: OK
        pg_basebackup compatible: OK
        systemid coherence: OK
        pg_receivexlog: OK
        pg_receivexlog compatible: OK
        receive-wal running: OK
        archiver errors: OK
```
### 4. Проверка синхронизации данных

```
-- На мастере:
CREATE TABLE test_replication (id serial, created_at timestamp, data text);
INSERT INTO test_replication (created_at, data) 
VALUES (now(), 'Test data from master 2025-05-15');

-- На реплике:
SELECT * FROM test_replication;

-- Результат:
 id |         created_at         |             data             
----+----------------------------+------------------------------
  1 | 2025-05-15 14:35:22.123456 | Test data from master 2025-05-15
(1 row)
```

### 5. Проверка слотов репликации

```
# На мастере:
psql -c "SELECT slot_name, active, restart_lsn FROM pg_replication_slots;"

# Результат:
    slot_name    | active | restart_lsn 
-----------------+--------+-------------
 standby_slot_1   | t      | 0/7000000
 barman          | t      | 0/7000000
(2 rows)
```

### 6. Проверка архивации WAL

```
# На мастере:
ls -lh /var/lib/postgresql/wal_archive

# Результат:
total 1.2G
-rw------- 1 postgres postgres 16M May 15 14:30 000000010000000000000001
-rw------- 1 postgres postgres 16M May 15 14:31 000000010000000000000002
```
