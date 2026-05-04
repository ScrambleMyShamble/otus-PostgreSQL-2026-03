# Блокировки
## Цели
* Понять как работают блокировки
* Научиться находить проблемные места

## Окружение
* Docker контейнер с Postgresql 17
* Vmvare с ОС ubuntu

## 1. Настройка
Логирование блокировок > 200 мс
```sql
psql (17.9 (Debian 17.9-1.pgdg13+1))
Type "help" for help.

postgres=# ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET deadlock_timeout = '200ms';
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)
```

Убедимся в применении настроек, првоерим логи
```bash
2026-05-04 11:04:31.204 UTC [1] LOG:  parameter "log_lock_waits" changed to "on"
2026-05-04 11:04:31.204 UTC [1] LOG:  parameter "deadlock_timeout" changed to "200ms"
```

Параметр deadlock_timeout определяет порог, после которого ожидание блокировки логируется как долгое.

## 2. Воспроизведение ожидания блокировки
Используем таблицу из предыдущих заданий

Сессия 1
```sql
BEGIN;
UPDATE test_table SET col = 1 WHERE id = 1;
```
и не коммитим
```sql
Сессия 2
BEGIN;
UPDATE test_table SET data = 'B' WHERE id = 1;
```

В логах контейнера Postgresql видим следующее
```bash
2026-05-04 11:14:16.069 UTC [642] LOG:  process 642 still waiting for ShareLock on transaction 2289735 after 200.280 ms
2026-05-04 11:14:16.069 UTC [642] DETAIL:  Process holding the lock: 274. Wait queue: 642.
```

## 3. Открыть три сессии psql к одной БД
Создадим новую таблицу и добавим запись

```sql
postgres=*# CREATE TABLE lock_test (id INT PRIMARY KEY, data TEXT);
CREATE TABLE
postgres=*# INSERT INTO lock_test VALUES (1, 'initial');
INSERT 0 1
```

### 3.1. Выполнить UPDATE в трёх сессиях (одна держит, две ждут)
Сессия A (держит блокировку)
```sql
BEGIN;
UPDATE lock_test SET data = 'A' WHERE id = 1;
```

Сессия B (ждёт)
```sql
BEGIN;
UPDATE lock_test SET data = 'B' WHERE id = 1;
```

Сессия C (ждёт)
```sql
BEGIN;
UPDATE lock_test SET data = 'B' WHERE id = 1;
```

### 3.2. Снять список блокировок из pg_locks. Для этого открываем 4 сессию.
```sql
postgres=# SELECT 
    locktype, 
    relation::regclass, 
    transactionid, 
    mode, 
    granted,
    pid,
    virtualtransaction
FROM pg_locks
WHERE NOT granted OR (locktype IN ('relation', 'transactionid'))
ORDER BY pid, granted;
   locktype    |    relation    | transactionid |       mode       | granted | pid  | virtualtransaction 
---------------+----------------+---------------+------------------+---------+------+--------------------
 transactionid |                |       2289737 | ExclusiveLock    | t       |  274 | 67/11
 relation      | lock_test_pkey |               | RowExclusiveLock | t       |  274 | 67/11
 relation      | lock_test      |               | RowExclusiveLock | t       |  274 | 67/11
 transactionid |                |       2289737 | ShareLock        | f       |  642 | 20/7
 relation      | lock_test      |               | RowExclusiveLock | t       |  642 | 20/7
 transactionid |                |       2289738 | ExclusiveLock    | t       |  642 | 20/7
 relation      | lock_test_pkey |               | RowExclusiveLock | t       |  642 | 20/7
 tuple         | lock_test      |               | ExclusiveLock    | f       | 1321 | 21/11
 relation      | lock_test_pkey |               | RowExclusiveLock | t       | 1321 | 21/11
 transactionid |                |       2289739 | ExclusiveLock    | t       | 1321 | 21/11
 relation      | lock_test      |               | RowExclusiveLock | t       | 1321 | 21/11
 relation      | pg_locks       |               | AccessShareLock  | t       | 1481 | 33/12
(12 rows)
```

### 3.3. Объясните смысл каждой блокировки из pg_locks

* PID 274 (транзакция 2289737) -> держит ExclusiveLock на свою транзакцию -> держит RowExclusiveLock на таблицу -> ещё НЕ закоммитилась
* PID 642 (транзакция 2289738) -> ждёт ShareLock на транзакцию 2289737 ← ОСНОВНАЯ БЛОКИРОВКА -> держит свою ExclusiveLock на 2289738 -> держит RowExclusiveLock на таблицу
* PID 1321 (транзакция 2289739) -> ждёт ExclusiveLock на tuple (конкретную версию строки) -> держит свою ExclusiveLock на 2289739 -> держит RowExclusiveLock на таблицу


|locktype|	что блокируется|	mode	|granted=f означает|
| -------- | -------- | -------- | -------- |
|transactionid	ID| транзакции|	ShareLock / ExclusiveLock	|Ожидание завершения чужой транзакции|
|tuple|	конкретная версия строки	|ExclusiveLock|	Ожидание доступа к физической версии строки|
|relation|	таблица или индекс|	RowExclusiveLock / AccessShareLock|	Редко бывает granted=f (обычно при DDL)|


## 4. Воспроизведение взаимоблокировки трёх транзакций
Сессия 1
```sql
postgres=# BEGIN;
UPDATE lock_test SET data = '1' WHERE id = 1;
BEGIN
UPDATE 1
```

Сессия 2
```sql
postgres=# BEGIN;
UPDATE lock_test SET data = '2' WHERE id = 2;
BEGIN
UPDATE 1
```

Сессия 3
```sql
postgres=# BEGIN;
UPDATE lock_test SET data = '3' WHERE id = 3;
BEGIN
UPDATE 1
```

## 4.1 Теперь создадим цикл взаимоблокировки:
* Сессия 1 (уже держит id=1)
Теперь пытаемся обновить строку, которую держит Сессия 2
```sql
UPDATE lock_test SET data = '274_tries_2' WHERE id = 2;
```

* Сессия 2 (уже держит id=2)
Теперь пытаемся обновить строку, которую держит Сессия 3
```sql
UPDATE lock_test SET data = '642_tries_3' WHERE id = 3;
```

* Сессия 3 (уже держит id=3)
Теперь пытаемся обновить строку, которую держит Сессия 1
```sql
UPDATE lock_test SET data = '1321_tries_1' WHERE id = 1;
```

Результат после выполнения команды апдейта в 3 сесси:
```sql
ERROR:  deadlock detected
DETAIL:  Process 1321 waits for ShareLock on transaction 2289740; blocked by process 274.
Process 274 waits for ShareLock on transaction 2289742; blocked by process 642.
Process 642 waits for ShareLock on transaction 2289744; blocked by process 1321.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,3) in relation "lock_test"
```

* Сессия 1 ждёт Сессию 2
* Сессия 2 ждёт Сессию 3
* Сессия 3 ждёт Сессию 1

## 4.2. Разбираемся постфактум
Что именно можно узнать из журнала
* Граф ожидания
```sql
DETAIL:  Process 1321 waits for ShareLock on transaction 2289740; blocked by process 274.
Process 274 waits for ShareLock on transaction 2289742; blocked by process 642.
Process 642 waits for ShareLock on transaction 2289744; blocked by process 1321.
```

* Какие SQL-запросы выполнялись, через логи контейнера
```bash 
2026-05-04 11:45:45.896 UTC [1321] ERROR:  deadlock detected
2026-05-04 11:45:45.896 UTC [1321] DETAIL:  Process 1321 waits for ShareLock on transaction 2289740; blocked by process 274.
	Process 274 waits for ShareLock on transaction 2289742; blocked by process 642.
	Process 642 waits for ShareLock on transaction 2289744; blocked by process 1321.
	Process 1321: UPDATE lock_test SET data = '1321_tries_1' WHERE id = 1;
	Process 274: UPDATE lock_test SET data = '274_tries_2' WHERE id = 2;
	Process 642: UPDATE lock_test SET data = '642_tries_3' WHERE id = 3;
2026-05-04 11:45:45.896 UTC [1321] HINT:  See server log for query details.
2026-05-04 11:45:45.896 UTC [1321] CONTEXT:  while updating tuple (0,3) in relation "lock_test"
2026-05-04 11:45:45.896 UTC [1321] STATEMENT:  UPDATE lock_test SET data = '1321_tries_1' WHERE id = 1;
2026-05-04 11:45:45.897 UTC [642] LOG:  process 642 acquired ShareLock on transaction 2289744 after 13388.138 ms
2026-05-04 11:45:45.897 UTC [642] CONTEXT:  while updating tuple (0,8) in relation "lock_test"
2026-05-04 11:45:45.897 UTC [642] STATEMENT:  UPDATE lock_test SET data = '642_tries_3' WHERE id = 3;
```






