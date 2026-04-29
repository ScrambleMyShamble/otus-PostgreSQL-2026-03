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
Для удобства запишем вывод сразу в текстовый файл pgbench_result_async.txt
```bash
pgbench -c 8 -j 4 -T 600 pgbench_test > pgbench_result_async.txt 2>&1
```
Заглянем внурь файла после отработавшей нагрузки
```bash
postgres@2457467c1dd5:~$cat pgbench_result_async.txt 
pgbench (17.9 (Debian 17.9-1.pgdg13+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 684014
number of failed transactions: 0 (0.000%)
latency average = 7.017 ms
initial connection time = 37.942 ms
tps = 1140.065890 (without initial connection time)
```
Нас интересует показатель tps

## 4. Снимок после
Статистика WAL
```sql
postgres=# SELECT pg_current_wal_lsn() AS wal_lsn_after;
 wal_lsn_after 
---------------
 1/1932A248
(1 row)
```
Статистика контрольных точек
```sql
postgres=# select * FROM pg_stat_bgwriter;
 buffers_clean | maxwritten_clean | buffers_alloc |          stats_reset          
---------------+------------------+---------------+-------------------------------
          6872 |                6 |        325017 | 2026-04-15 17:25:52.906109+00
(1 row)
```
Количество WAL-файлов
```sql
postgres=# SELECT wal_records, wal_fpi, wal_bytes 
FROM pg_stat_wal;
 wal_records | wal_fpi | wal_bytes  
-------------+---------+------------
    38306743 |   50411 | 4591320646
(1 row)
```

## 5. Расчёт объёма WAL за 10 минут и средний объём на checkpoint

### 5.1 Объёма WAL высчитывается как разность wal_bytes из pg_stat_wal до и после
4591320646 - 4293187166 ≈ 284.33 МБ

### 5.2 Количество контрольных точек за 10 минут
Вычитываем из логов контейнера
```bash
docker logs 2457467c1dd5 2>&1 | grep "checkpoint starting" | wc -l
50
```

### 5.3 Время первой и последней КТ 
```bash
root@adminer-VMware-Virtual-Platform:/var/lib/postgresql/data# docker logs 2457467c1dd5 2>&1 | grep "checkpoint starting" | head -1
2026-04-18 15:48:30.241 UTC [27] LOG:  checkpoint starting: time
root@adminer-VMware-Virtual-Platform:/var/lib/postgresql/data# docker logs 2457467c1dd5 2>&1 | grep "checkpoint starting" | tail -1
2026-04-29 13:39:54.078 UTC [27] LOG:  checkpoint starting: time
root@adminer-VMware-Virtual-Platform:/var/lib/postgresql/data# 
```

### 5.4 Из логов postgresql можем посчитать статистику по КТ
Через команду
```docker
docker logs 2457467c1dd5 2>&1 | grep "checkpoint"
```





















(1 row)
  
