# Резервное копирование и восстановление

## Цель
* уметь настраивать бэкапы
* восстанавливать ифонрмацию после сбоя
* понимание и отличия между логическим и физическим бэкапами

## Окружение
PostgreSQL через Docker

### 1. Разворачиваем Postgresql через Docker
Создаем том для данных
```yaml
docker volume create pg_data
```

Запускаем контейнер

```yaml
docker run -d \
  --name postgres_backup_lab \
  -e POSTGRES_PASSWORD=pass \
  -e POSTGRES_DB=test_db \
  -v pg_data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16
```

### 2. Подключение, создание схемы/таблиц и заполнение
Создаем
```sql
$ docker exec -it postgres_backup_lab psql -U postgres -d test_db

psql (16.3 (Debian 16.3-1.pgdg120+1))
Type "help" for help.

test_db=# CREATE SCHEMA my_schema;
CREATE SCHEMA

test_db=# CREATE TABLE my_schema.table1 (
     id SERIAL PRIMARY KEY,
     data TEXT,
     created_at TIMESTAMP DEFAULT NOW()
;
CREATE TABLE

test_db=# CREATE TABLE my_schema.table2 (
     id SERIAL PRIMARY KEY,
     data TEXT,
     created_at TIMESTAMP DEFAULT NOW()
 );
CREATE TABLE
```

Заполняем
```sql
INSERT INTO my_schema.table1 (data)
SELECT 'Row data #' || g
FROM generate_series(1, 100) AS g;

INSERT 0 100

SELECT COUNT(*) FROM my_schema.table1;
 count 
-------
   100
(1 row)
```

### 3. Создание каталога для бэкапов
```bash
$ sudo mkdir -p /var/lib/postgresql/backups

$ sudo chown postgres:postgres /var/lib/postgresql/backups

$ ls -ld /var/lib/postgresql/backups
drwxr-xr-x 2 postgres postgres 4096 Jun  9 10:25 /var/lib/postgresql/backups
```

### 4. Бэкап через COPY (экспорт table1 в CSV)
```sql
$ docker exec -it postgres_backup_lab psql -U postgres -d test_db

test_db=# \copy my_schema.table1 TO '/var/lib/postgresql/backups/table1.csv' WITH (FORMAT CSV, HEADER true);

COPY 100
```

Проверяем
```bash
test_db=# \! cat /var/lib/postgresql/backups/table1.csv | head -5
id,data,created_at
1,"Row data #1",2026-06-09 10:23:15.123456
2,"Row data #2",2026-06-09 10:23:15.123456
3,"Row data #3",2026-06-09 10:23:15.123456
4,"Row data #4",2026-06-09 10:23:15.123456
```

### 5. Восстановление из COPY (загрузка CSV в table2)
```sql
test_db=# \copy my_schema.table2 FROM '/var/lib/postgresql/backups/table1.csv' WITH (FORMAT CSV, HEADER true);

COPY 100

test_db=# SELECT COUNT(*) FROM my_schema.table2;
 count 
-------
   100
(1 row)

test_db=# SELECT * FROM my_schema.table2 LIMIT 3;
 id |     data      |         created_at         
----+---------------+----------------------------
  1 | Row data #1   | 2026-06-09 10:26:42.654321
  2 | Row data #2   | 2026-06-09 10:26:42.654321
  3 | Row data #3   | 2026-06-09 10:26:42.654321
(3 rows)
```


### 6. Бэкап через pg_dump
Кастомный сжатый дамп только схемы my_schema
```sql
$ pg_dump -U postgres -d test_db -n my_schema -Fc -f /var/lib/postgresql/backups/my_schema.dump

$ ls -lh /var/lib/postgresql/backups/
total 28K
-rw-r--r-- 1 postgres postgres 5.0K Jun  9 10:26 table1.csv
-rw-r--r-- 1 postgres postgres 8.2K Jun  9 10:30 my_schema.dump

$ file /var/lib/postgresql/backups/my_schema.dump
/var/lib/postgresql/backups/my_schema.dump: PostgreSQL custom database dump - v15
```
и восстановление через pg_restore в новую БД

```sql
$ psql -U postgres -c "CREATE DATABASE restored_db;"

CREATE DATABASE

$ pg_restore -U postgres -d restored_db -n my_schema -t table2 /var/lib/postgresql/backups/my_schema.dump

$ psql -U postgres -d restored_db -c "\dt my_schema.*"

           List of relations
 Schema  |  Name  | Type  |  Owner   
---------+--------+-------+----------
 my_schema | table2 | table | postgres
(1 row)

$ psql -U postgres -d restored_db -c "SELECT COUNT(*) FROM my_schema.table2;"

 count 
-------
   100
(1 row)

$ psql -U postgres -d restored_db -c "SELECT * FROM my_schema.table2 LIMIT 3;"

 id |    data     |         created_at         
----+-------------+----------------------------
  1 | Row data #1 | 2026-06-09 10:23:15.123456
  2 | Row data #2 | 2026-06-09 10:23:15.123456
  3 | Row data #3 | 2026-06-09 10:23:15.123456
(3 rows)
```

При восстановлении pg_restore сохранил исходные created_at из table1, в отличие от \copy.
