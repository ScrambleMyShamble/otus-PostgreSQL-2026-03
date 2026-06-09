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
``` bash
$ sudo mkdir -p /var/lib/postgresql/backups

$ sudo chown postgres:postgres /var/lib/postgresql/backups

$ ls -ld /var/lib/postgresql/backups
drwxr-xr-x 2 postgres postgres 4096 Jun  9 10:25 /var/lib/postgresql/backups
```



