# Журналы

## Цели и задания
* Настройка журналирования
* Настрйока контрольных точек
* Наглядно увидеть влияние настроек на проиводительность кластера и объем занимаемого места журналом.

## 1. Настройка конфигурации и pgbench
```sql
root@2457467c1dd5:/# pgbench --version
pgbench (PostgreSQL) 17.9 (Debian 17.9-1.pgdg13+1)
```

Меняем настройки и передергиваем конфигурационный файл
```sql
postgres=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)

postgres=# show checkpoint_timeout;
 checkpoint_timeout 
--------------------
 30s
(1 row)

postgres=# show log_checkpoints;
 log_checkpoints 
-----------------
 on
(1 row)
```

Инициализируем тестовую БД и гоним pgbench
```sql
postgres=# create database pgbench_test;
CREATE DATABASE

pgbench -i -s 100 pgbench_test

postgres@2457467c1dd5:~$ pgbench -i -s 100 pgbench_test
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
vacuuming...                                                                                   
creating primary keys...
done in 62.96 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 48.02 s, vacuum 0.62 s, primary keys 14.30 s).
```


## 2. Снимок ДО 

Статистика контрольных точек
```sql
postgres=# SELECT * FROM pg_stat_bgwriter;
 buffers_clean | maxwritten_clean | buffers_alloc |          stats_reset          
---------------+------------------+---------------+-------------------------------
          6872 |                6 |        131372 | 2026-04-15 17:25:52.906109+00
```

Статистика WAL
```sql
postgres=# SELECT pg_current_wal_lsn() AS wal_lsn_before;
 wal_lsn_before 
----------------
 1/6C9DF50
(1 row)
```

Также можно посмотреть количество WAL-файлов (сегментов) сброшено
```sql
postgres=# SELECT wal_records, wal_fpi, wal_bytes 
FROM pg_stat_wal;
 wal_records | wal_fpi | wal_bytes  
-------------+---------+------------
    33995145 |   50411 | 4293187166
(1 row)
```

## 3. Запускаем нагрузку на 10 минут и собираем статистику После
```bash
pgbench -c 8 -j 4 -T 600 pgbench_test > pgbench_result_async.txt 2>&1
```












 on
(1 row)
  
