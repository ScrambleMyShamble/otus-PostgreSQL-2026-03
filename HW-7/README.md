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
вытаскиваем все логи(файл 'Postgres log' приложен) связанные с checkpoint и считаем(уточню, время выставленное на сервере ubuntu не совпадает с Московским, небольшой рассинхрон в 3часа)

* Количество КТ за 10 минут:
21 КТ (с 13:29:54 по 13:39:54 включительно)

* Объём WAL:
Вы указали ранее: 284.33 МБ

* Средний WAL на КТ:
284.33 / 21 = 13.54 МБ

* Выполнение по расписанию?
Интервал между стартами: 30 секунд

|Показатель	|Значение|
| -------- | -------- |
|TPS (асинхронный)|	1140|
|Объём WAL за 10 мин	|284.33 МБ|
|Количество КТ|	21|
|Средний| WAL/КТ	13.54 МБ|
|Соблюдение расписания|	Да (30 сек)|


## 6. Сравнение TPS при синхронном и асинхронном коммите
### 6.1 Прогон 1 (синхронный, стандартный)
```sql
postgres=# ALTER SYSTEM SET synchronous_commit = on;
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)
```

```sql
postgres@2457467c1dd5:/var/lib$ pgbench -c 8 -j 4 -T 600 pgbench_test
pgbench (17.9 (Debian 17.9-1.pgdg13+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 686974
number of failed transactions: 0 (0.000%)
latency average = 6.987 ms
initial connection time = 42.340 ms
tps = 1145.010106 (without initial connection time)
```

### 6.2 Прогон 2 (асинхронный)
```sql
postgres=# ALTER SYSTEM SET synchronous_commit = off;
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)
```

```sql
postgres@2457467c1dd5:/var/lib$ pgbench -c 8 -j 4 -T 600 pgbench_test
pgbench (17.9 (Debian 17.9-1.pgdg13+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 720389
number of failed transactions: 0 (0.000%)
latency average = 6.663 ms
initial connection time = 39.177 ms
tps = 1200.684968 (without initial connection time)
```

Видим небольшую разницу в tps, около 5%. Маленькая разница обусловлена высокопроизводительным диском.

Даже при небольшой разнице в моем тесте, нужно учитывать что:
* Синхронный коммит – при каждом COMMIT транзакции лидер ждёт, пока WAL-записи сбросятся на диск (на master + синхронные реплики, если есть).
* Асинхронный коммит – транзакция считается зафиксированной, как только WAL записан в буфер ОС (или память), а сброс на диск происходит позже.
* Цена синхронности – задержка каждого коммита = минимум время одного fsync (миллисекунды), что при коротких транзакциях (pgbench по умолчанию – 1 INSERT/UPDATE) резко снижает TPS.


## 7. Задание со звездочкой
Создадим отдельный контейнер под это задание
<img width="1495" height="345" alt="image" src="https://github.com/user-attachments/assets/3ee6f57f-d94f-46c5-b5f0-6b44304c7b77" />

Инициализируем кластер с контрольными суммами
```sql
postgres@2457467c1dd5:/$ initdb -D /var/lib/postgresql/checksum_cluster --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "C.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

creating directory /var/lib/postgresql/checksum_cluster ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default "max_connections" ... 100
selecting default "shared_buffers" ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

initdb: warning: enabling "trust" authentication for local connections
initdb: hint: You can change this by editing pg_hba.conf or using the option -A, or --auth-local and --auth-host, the next time you run initdb.

Success. You can now start the database server using:

    pg_ctl -D /var/lib/postgresql/checksum_cluster -l logfile start
```

Проверяем включение сумм
```sql
postgres=# SHOW data_checksums;
 data_checksums 
----------------
 on
(1 row)
```

### 7.1. Создаем таблицу и заполняем данными
```sql
CREATE TABLE test_table (
    id SERIAL PRIMARY KEY,
    data TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO test_table (data) 
SELECT 'Test data ' || generate_series(1, 10000);
INSERT 0 10000

postgres=# SELECT COUNT(*) FROM test_table;
 count 
-------
 10000
(1 row)
```

Смотрим в каком файле сохранена таблица
```sql
postgres=# SELECT pg_relation_filepath('test_table');
 pg_relation_filepath 
----------------------
 base/5/41056
(1 row)
```

### 7.2. Имитация повреждения файла таблицы
Останавливаем кластер
```sql
postgres@2457467c1dd5:/$ pg_ctl stop -D /var/lib/postgresql/data
waiting for server to shut down....2026-04-29 15:37:52.668 UTC [7166] LOG:  received fast shutdown request
2026-04-29 15:37:52.670 UTC [7166] LOG:  aborting any active transactions
2026-04-29 15:37:52.671 UTC [7259] FATAL:  terminating connection due to administrator command
2026-04-29 15:37:52.685 UTC [7166] LOG:  background worker "logical replication launcher" (PID 7172) exited with exit code 1
2026-04-29 15:37:52.688 UTC [7167] LOG:  shutting down
2026-04-29 15:37:52.691 UTC [7167] LOG:  checkpoint starting: shutdown immediate
2026-04-29 15:37:52.743 UTC [7167] LOG:  checkpoint complete: wrote 120 buffers (0.7%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.006 s, sync=0.033 s, total=0.056 s; sync files=15, longest=0.013 s, average=0.003 s; distance=1612 kB, estimate=1612 kB; lsn=0/1943678, redo lsn=0/1943678
2026-04-29 15:37:52.757 UTC [7166] LOG:  database system is shut down
 done
server stopped
```

Повреждаем файл
```sql
postgres@1163dbb179e0:~/data/base/5$ dd if=/dev/zero of=41056 bs=100 count=1 conv=notrunc
1+0 records in
1+0 records out
100 bytes copied, 0.000356562 s, 280 kB/s
```

```bash
WARNING:  page verification failed, calculated checksum mismatch
ERROR:  invalid page header in block 0 of relation base/5/41056
```

### 7.3. Корректные варианты действий для продолжения работы
```sql
SET ignore_checksum_failure = on;
SET zero_damaged_pages = on;
SELECT COUNT(*) FROM test_table;
WARNING:  page verification failed, calculated checksum mismatch
ERROR:  invalid page header in block 0 of relation base/5/41056
```
### 7.3.1. Извлечь данные не удалось, так как повреждена первая страница

### 7.3.2. Создание резервной копии неповреждённых таблиц
```sql
pg_dump -U postgres -t test_table2 > test_table2_backup.sql
pg_dump: warning: skipping table "test_table" because it does not exist
```

### 7.3.3. Удаление повреждённой таблицы
```sql
DROP TABLE test_table;
DROP TABLE
```

Проверка
```sql
SELECT table_name FROM information_schema.tables WHERE table_name = 'test_table';
 table_name 
------------
(0 rows)
```

### 7.3.4. Восстановление таблицы из резервной копии
```sql
pg_restore -U postgres -d postgres test_table_backup.dump
pg_restore: finished successfully
```

```sql
SELECT COUNT(*) FROM test_table;
 count 
-------
  1000
(1 row)
```
(1 row)
  
