## 1. Подготовка базы

Запускаем контейнер и подключаемся к PostgreSQL:

```bash
docker exec -it psql bash
su - postgres
psql
```

Создаём БД:

```sql
CREATE DATABASE appdb;
```

Подключаемся к ней:

```sql
\c appdb
```

Создаём таблицу:

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    balance NUMERIC(10,2) NOT NULL
);
```

Добавляем данные:

```sql
INSERT INTO users (name, email, balance) 
VALUES 
    ('Roman', 'mail@gmail.com', 5000), 
    ('Lera', 'mail2@gmail.com', 100), 
    ('John', 'mail3@gmail.com', 1000000);
```

Проверяем:

```sql
SELECT * FROM users;
```

Результат:

```text
 id | name  |      email      |  balance
----+-------+-----------------+------------
  1 | Roman | mail@gmail.com  |    5000.00
  2 | Lera  | mail2@gmail.com |     100.00
  3 | John  | mail3@gmail.com | 1000000.00
```

---

# 2. Подготовка WAL Archiving

Узнаём расположение `PGDATA`:

```sql
SHOW data_directory;
```

Результат:

```text
/var/lib/postgresql/data
```

Создаём каталог для архивных WAL:

```bash
mkdir -p /var/lib/postgresql/wal_archive
chown postgres:postgres /var/lib/postgresql/wal_archive
```

Проверяем текущие настройки:

```sql
SHOW wal_level;
SHOW archive_mode;
SHOW archive_command;
```

Было:

```text
wal_level      = replica
archive_mode   = off
archive_command = (disabled)
```

`wal_level = replica` уже подходит для физической репликации и PITR.

Включаем архивирование:

```sql
ALTER SYSTEM SET archive_mode = 'on';
ALTER SYSTEM SET archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f';
```

`archive_mode` требует **полного restart PostgreSQL**.

Перезапускаем под пользователем `postgres`:

```bash
su - postgres

/usr/lib/postgresql/17/bin/pg_ctl \
    -D /var/lib/postgresql/data \
    restart
```

Проверяем:

```sql
SHOW archive_mode;
SHOW archive_command;
```

Теперь:

```text
archive_mode = on

archive_command =
cp %p /var/lib/postgresql/wal_archive/%f
```

Проверяем статус архивации:

```sql
SELECT
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count,
    last_failed_wal,
    last_failed_time
FROM pg_stat_archiver;
```

`failed_count = 0` означает, что архивирование не падает.

Для теста вручную переключаем WAL:

```sql
SELECT pg_switch_wal();
```

После этого в `wal_archive` появляется WAL-сегмент.

Проверяем:

```bash
ls -lah /var/lib/postgresql/wal_archive
```

---

# 3. Physical Base Backup

Для PITR нужен **physical base backup**, а не `pg_dump`.

Создаём его:

```bash
pg_basebackup \
    -h localhost \
    -U postgres \
    -D /var/lib/postgresql/base_backup \
    -Fp \
    -Xs \
    -P \
    -v
```

Основные параметры:

```text
-Fp  → plain format, обычная структура PGDATA
-Xs  → WAL передаётся через streaming во время backup
-P   → прогресс
-v   → подробный вывод
```

После выполнения `base_backup` содержит полноценный PostgreSQL `$PGDATA`:

```text
base_backup/
├── base/
├── global/
├── pg_wal/
├── PG_VERSION
├── postgresql.conf
├── pg_hba.conf
└── ...
```

Этот backup можно использовать как основу для запуска отдельного PostgreSQL-инстанса.

---

# 4. Изменения после Base Backup

После создания backup делаем изменения, которых **ещё нет в base backup**, но они попадут в WAL.

Добавляем пользователя:

```sql
SELECT now();

INSERT INTO users (name, email, balance)
VALUES ('Alice', 'alice@gmail.com', 2500);
```

Изменяем баланс:

```sql
UPDATE users
SET balance = 9999
WHERE name = 'Roman';

SELECT now();
```

Получили:

```text
09:40:18  Alice INSERT
09:40:30  Roman balance → 9999
```

Теперь создаём ошибку:

```sql
DELETE FROM users
WHERE name = 'Roman';

SELECT now();
```

Ошибка произошла:

```text
09:40:43.952...
```

После этого принудительно переключаем WAL:

```sql
SELECT pg_switch_wal();
```

Проверяем архивирование:

```sql
SELECT
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count
FROM pg_stat_archiver;
```

---

# 5. PITR — Point-in-Time Recovery

Цель:

> Восстановить базу к моменту **до ошибочного DELETE**.

Имеем:

```text
Base backup
    ↓
WAL
    ↓
09:40:18  INSERT Alice
    ↓
09:40:30  UPDATE Roman → 9999
    ↓
09:40:43  DELETE Roman  ← ошибка
```

Создаём отдельный recovery PostgreSQL.

`base_backup` монтируется как его `$PGDATA`:

```yaml
postgresql-recovery:
  container_name: psql-recovery
  image: postgres:17
  environment:
    POSTGRES_PASSWORD: pwd
  volumes:
    - ./base_backup:/var/lib/postgresql/data
    - ./wal_archive:/var/lib/postgresql/wal_archive
```

Важно:

**Recovery-инстанс должен использовать существующий `base_backup`, а не создавать новый кластер через `initdb`.**

---

## Recovery configuration

В:

```text
base_backup/postgresql.conf
```

добавляем:

```conf
restore_command = 'cp /var/lib/postgresql/wal_archive/%f %p'

recovery_target_time = '2026-08-28 09:40:43'
```

`restore_command`:

> говорит PostgreSQL, откуда брать необходимые WAL.

`recovery_target_time`:

> говорит PostgreSQL, до какого момента проигрывать WAL.

Создаём:

```bash
touch ./base_backup/recovery.signal
```

`recovery.signal` сообщает PostgreSQL:

> этот кластер должен запускаться в recovery-режиме.

---

# 6. Запуск PITR

Запускаем recovery:

```bash
docker compose up -d postgresql-recovery
```

В логах увидели:

```text
starting point-in-time recovery to 2026-08-28 09:40:43+00
```

PostgreSQL:

1. взял base backup;
2. нашёл нужные WAL через `restore_command`;
3. начал replay WAL;
4. дошёл до указанного времени;
5. остановился **до транзакции DELETE**.

В логах:

```text
recovery stopping before commit of transaction ...
time 2026-08-28 09:40:43.951449+00
```

---

# 7. Проверка восстановленной базы

Подключаемся к recovery PostgreSQL:

```bash
docker exec -it psql-recovery bash
su - postgres
psql
```

Проверяем:

```sql
SELECT * FROM users;
```

Получаем:

```text
 id | name  |      email      |  balance
----+-------+-----------------+------------
  1 | Roman | mail@gmail.com  |    9999.00
  2 | Lera  | mail2@gmail.com |     100.00
  3 | John  | mail3@gmail.com | 1000000.00
  4 | Alice | alice@gmail.com |    2500.00
```

То есть:

```text
Alice INSERT - есть       
Roman UPDATE - есть      
Roman DELETE - нет      
```

Таким образом PITR восстановил состояние **до ошибочного DELETE**.

Проверяем:

```sql
SELECT pg_is_in_recovery();
```

Результат:

```text
t
```

Это означает, что PostgreSQL всё ещё находится в recovery и работает read-only.

---

# 8. Promotion

Когда убеждаемся, что восстановленное состояние подходит, делаем promotion:

```sql
SELECT pg_promote();
```

Проверяем:

```sql
SELECT pg_is_in_recovery();
```

Результат:

```text
f
```

Теперь PostgreSQL стал обычным **read-write** сервером.

Проверяем запись:

```sql
INSERT INTO users (name, email, balance)
VALUES ('Test', 'test@gmail.com', 100);
```

---

# 9. Главное различие `pg_dump` и `pg_basebackup`

## `pg_dump`

Logical backup:

```text
PostgreSQL
    ↓
pg_dump
    ↓
backup.dump
```

Содержит логическое представление базы:

* таблицы;
* данные;
* индексы;
* constraints;
* другие объекты.

Восстанавливается через:

```bash
pg_restore
```

Сам `.dump` не является `$PGDATA` и не может быть просто запущен как PostgreSQL.

`pg_dump` возвращает базу **к состоянию на момент создания dump**.

---

## `pg_basebackup`

Physical backup:

```text
PostgreSQL
    ↓
pg_basebackup
    ↓
PGDATA
```

Это физическая копия PostgreSQL-кластера.

Её можно использовать как основу для запуска PostgreSQL.

Именно:

```text
pg_basebackup + WAL
```

используются для PITR.

---

# 10. WAL

WAL = **Write-Ahead Log**.

Упрощённо:

```text
SQL operation
      ↓
     WAL
      ↓
data pages
```

WAL содержит информацию, необходимую PostgreSQL для восстановления изменений.

Локально находится в:

```text
$PGDATA/pg_wal/
```

Для PITR WAL необходимо архивировать во внешнее/отдельное хранилище:

```text
PostgreSQL
    ↓
WAL
    ↓
WAL archive
```

В лаборатории использовали:

```text
/var/lib/postgresql/wal_archive
```

В production это может быть:

```text
S3
MinIO
NFS
backup server
```

---

# 11. RPO / RTO

### RPO — Recovery Point Objective

Отвечает на вопрос:

> **Сколько данных мы можем потерять?**

Например:

```text
RPO = 5 минут
```

означает:

> При аварии допускается потеря максимум 5 минут данных.

Это не означает, что мы специально теряем 5 минут.

При правильном WAL archiving фактический RPO может быть, например, значительно меньше.

---

### RTO — Recovery Time Objective

Отвечает на вопрос:

> **Сколько времени максимально может занимать восстановление?**

Например:

```text
RTO = 30 минут
```

означает:

```text
авария
  ↓
recovery
  ↓
сервис снова работает
```

и весь процесс должен уложиться максимум в 30 минут.

Поэтому RTO нужно проверять реальным recovery-тестом.

---

# 12. Replication / HA

Реплика — это **отдельный работающий PostgreSQL-инстанс**, поддерживаемый в актуальном состоянии с Primary.

Схема:

```text
Primary
   |
   | WAL streaming
   v
Replica
```

На Primary:

```text
walsender
```

На Replica:

```text
walreceiver
```

WAL передаётся по сети напрямую:

```text
Primary
   |
 TCP connection
   |
   v
Replica
```

Общее хранилище для этого не требуется.

Упрощённая схема:

```text
Primary
   |
   +----→ Replica      ← HA / failover
   |
   +----→ S3           ← Backup / PITR
```

---

# 13. Replication ≠ Backup

Репликация защищает прежде всего от:

```text
server failure
VM failure
disk failure
```

Например:

```text
Primary
    ↓
Replica
    ↓
promote
    ↓
New Primary
```

Но если на Primary выполнить:

```sql
DELETE FROM users;
COMMIT;
```

то этот `DELETE` попадёт в WAL и будет применён на Replica.

Получается:

```text
Primary
   |
   | DELETE
   | WAL
   ↓
Replica
   |
   ↓
DELETE тоже выполнен
```

Поэтому:

```text
Replication → HA / доступность

Backup + PITR → восстановление старого состояния данных
```

На production обычно используют **оба механизма одновременно**.

---

# 14. Что запомнить как DevOps

```text
pg_dump
→ logical backup
→ pg_restore

pg_basebackup
→ physical backup
→ основа для PITR

WAL
→ журнал изменений PostgreSQL

WAL archiving
→ хранение WAL вне сервера

PITR
→ восстановление на конкретный момент времени

pg_promote()
→ завершение recovery и переход в read-write

RPO
→ сколько данных допустимо потерять

RTO
→ сколько времени допустимо восстанавливаться

Replication
→ HA / быстрый failover
```

Главная production-модель:

```text
                         ┌───────────────┐
                         │    Replica    │
                         │   HA/failover │
                         └───────▲───────┘
                                 │
                                 │ WAL
                                 │
                         ┌───────┴───────┐
                         │    Primary    │
                         │  PostgreSQL   │
                         └───────┬───────┘
                                 │
                                 │ WAL archive
                                 ▼
                         ┌───────────────┐
                         │ S3 / Storage  │
                         │ Backup + PITR │
                         └───────────────┘
```

И ещё одна важная мысль:

> **Application rollback не обязательно означает Database rollback.**

В production часто стараются сделать migrations backward-compatible, чтобы можно было:

```text
Application v2 → rollback → Application v1
```

при этом оставить:

```text
Database schema v2
```

и не рисковать данными откатом схемы.