# Работа vacuum и autovacuum в Postgresql 17.
## Задание и цели
* Понять как работает vacuum и autovacuum, их отличие
* Что такое многоверсионность и как она влияет на использование места на диске

## 1. Окружение
* Vmvare с ОС ubuntu
* Docker rонтейнер с Postgresql 17
* Утилита pgbench для прогона тестов
* Подклчюченный к виртуальной машине Dbeaver

## 2. Подготовка БД через pgbench

```sql
root@2457467c1dd5:/# su - postgres -c "pgbench -i postgres"
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
vacuuming...                                                                              
creating primary keys...
done in 0.46 s (drop tables 0.01 s, create tables 0.02 s, client-side generate 0.23 s, vacuum 0.10 s, primary keys 0.09 s).
```

И сразу прогон теста
```sql
pgbench -c8 -P 6 -T 60 -U postgres postgres

root@2457467c1dd5:/# pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (17.9 (Debian 17.9-1.pgdg13+1))
starting vacuum...end.
progress: 6.0 s, 818.7 tps, lat 9.613 ms stddev 5.360, 0 failed
progress: 12.0 s, 838.2 tps, lat 9.511 ms stddev 5.015, 0 failed
progress: 18.0 s, 745.1 tps, lat 10.696 ms stddev 7.167, 0 failed
progress: 24.0 s, 808.2 tps, lat 9.866 ms stddev 5.660, 0 failed
progress: 30.0 s, 812.1 tps, lat 9.823 ms stddev 5.326, 0 failed
progress: 36.0 s, 821.2 tps, lat 9.709 ms stddev 4.984, 0 failed
progress: 42.0 s, 855.6 tps, lat 9.317 ms stddev 4.833, 0 failed
progress: 48.0 s, 875.0 tps, lat 9.109 ms stddev 4.619, 0 failed
progress: 54.0 s, 874.8 tps, lat 9.112 ms stddev 4.687, 0 failed
progress: 60.0 s, 876.2 tps, lat 9.098 ms stddev 4.635, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 49958
number of failed transactions: 0 (0.000%)
latency average = 9.565 ms
latency stddev = 5.263 ms
initial connection time = 72.424 ms
tps = 833.415647 (without initial connection time)
```

## 3. Состояние обслуживания после нагрузки
```sql
root@2457467c1dd5:/# psql -U postgres
psql (17.9 (Debian 17.9-1.pgdg13+1))
Type "help" for help.

postgres=# SELECT relname, n_dead_tup, last_autovacuum, last_vacuum
FROM pg_stat_user_tables;
     relname      | n_dead_tup |        last_autovacuum        |          last_vacuum          
------------------+------------+-------------------------------+-------------------------------
 orders           |          0 |                               | 
 pgbench_branches |          0 | 2026-04-25 12:44:25.381494+00 | 2026-04-25 12:42:57.050696+00
 pgbench_accounts |       3941 | 2026-04-25 12:41:25.137755+00 | 2026-04-25 12:40:50.0564+00
 pgbench_history  |          0 | 2026-04-25 12:44:25.392302+00 | 2026-04-25 12:40:50.135933+00
 pgbench_tellers  |          0 | 2026-04-25 12:44:25.382434+00 | 2026-04-25 12:42:57.051462+00
 t2               |          0 |                               | 
```

## 4. Работа с тестовой таблицей
### 4.1 Создаем таблицу и заполняем 1 млн строк
```sql
postgres=# CREATE TABLE test_table (
    id serial PRIMARY KEY,
    text_field text
);
CREATE TABLE
postgres=# INSERT INTO test_table (text_field)
SELECT 'test text ' || generate_series(1, 1000000);
INSERT 0 1000000
```

### 4.2 Фиксируем размер таблицы
```sql
postgres=# SELECT pg_size_pretty(pg_total_relation_size('test_table')) AS total_size;
 total_size 
------------
 71 MB
(1 row)
```

### 4.3 и количество мертвых строк
```sql
postgres=# SELECT relname, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'test_table';
  relname   | n_dead_tup 
------------+------------
 test_table |          0
(1 row)
```
ответ 0 - таблица актуальная, и так как никаких изменения в ней еще не происходило, логично что мертвых строк в ней быть не может.

