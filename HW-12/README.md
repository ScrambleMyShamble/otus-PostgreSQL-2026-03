# Виды и устройство репликации в PostgreSQL. Практика применения

## Цель.
* знать, когда и где использовать репликацию
* понимать возможности каждого способоа
* выбирать оптимальны способ, в зависимости от задачи

### Релизовываем свой кластер из 3 виртуальных машин
Использовать буду vmvare. Так как уже есть 4 виртуальные машины, которые я создавал для домашних проектов, создавать и настраивать их уже не нужно, начинаю сразу
с настройки Postgresql на каждой вм(далее называю их вм1, вм2, вм3)

### 1. Настройка ВМ1:
Запускаем постгрю, создаем нужные объекты
```bash
$ sudo systemctl start postgresql
$ sudo -u postgres psql
```

```sql
postgres=# CREATE DATABASE replication_lab;
CREATE DATABASE

postgres=# \c replication_lab
You are now connected to database "replication_lab" as user "postgres".

replication_lab=# CREATE TABLE test (
    id SERIAL PRIMARY KEY,
    data TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
CREATE TABLE

replication_lab=# CREATE TABLE test2 (
    id SERIAL PRIMARY KEY,
    data TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
CREATE TABLE
```

### 1.2 Настройка публикации
```sql
replication_lab=# CREATE PUBLICATION pub_test FOR TABLE test;
CREATE PUBLICATION

replication_lab=# \dRp+
                          Publication pub_test
  Owner   | All tables | Inserts | Updates | Deletes | Truncates 
----------+------------+---------+---------+---------+-----------
 postgres | f          | t       | t       | t       | t
Tables:
    "public.test"
```

### 1.3 Настройка параметров PostgreSQL на ВМ1
```bash
sudo nano /etc/postgresql/16/main/postgresql.conf
```
Интересуют нас следующие параметры
listen_addresses = '*'
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10

и рестартуем кластер
```bash
sudo systemctl restart postgresql
```

проверяем
```bash
$ sudo -u postgres psql -c "SHOW wal_level;"
 wal_level 
-----------
 logical
(1 row)
```

### 1.4 Добавление правила в pg_hba.conf на ВМ1
```bash
sudo nano /etc/postgresql/16/main/pg_hba.conf
```
host    replication     all             10.0.0.0/24
host    all             all             10.0.0.0/24

### 2. НАСТРОЙКА ВМ2
```sql
postgres=# CREATE DATABASE replication_lab;
CREATE DATABASE

postgres=# \c replication_lab
You are now connected to database "replication_lab" as user "postgres".

replication_lab=# CREATE TABLE test2 (
    id SERIAL PRIMARY KEY,
    data TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
CREATE TABLE

replication_lab=# CREATE TABLE test (
    id SERIAL PRIMARY KEY,
    data TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
CREATE TABLE
```
### 2.1 Настройка публикации таблицы test2 на ВМ2
```sql
replication_lab=# CREATE PUBLICATION pub_test2 FOR TABLE test2;
CREATE PUBLICATION

replication_lab=# \dRp+
                          Publication pub_test2
  Owner   | All tables | Inserts | Updates | Deletes | Truncates 
----------+------------+---------+---------+---------+-----------
 postgres | f          | t       | t       | t       | t
Tables:
    "public.test2"
```

### 2.2 Создание подписки на ВМ2 для получения test с ВМ1
```sql
replication_lab=# CREATE SUBSCRIPTION sub_test_from_vm1
CONNECTION 'host=10.0.0.1 port=5432 dbname=replication_lab user=postgres password=strongpass'
PUBLICATION pub_test
WITH (create_slot = true, enabled = true);
NOTICE:  created replication slot "sub_test_from_vm1" on publisher
CREATE SUBSCRIPTION

replication_lab=# \dRs+
                        List of subscriptions
        Name         |  Owner   | Enabled | Publication |   Connstring   
---------------------+----------+---------+-------------+----------------
 sub_test_from_vm1   | postgres | t       | {pub_test}  | host=10.0.0.1...
(1 row)
```





















