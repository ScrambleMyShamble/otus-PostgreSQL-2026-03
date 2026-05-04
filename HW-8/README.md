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

Параметр deadlock_timeout определяет порог, после которого ожидание блокировки логируется как долгое.

## 2. Воспроизведение ожидания блокировки
Пересоздадим таблицу, вместо старой, из прошлого урока
```sql
postgres=# CREATE TABLE test_table (
    id INTEGER PRIMARY KEY,
    data TEXT
);
CREATE TABLE
postgres=# INSERT INTO test_table (id, data) VALUES (1, 'initial');
INSERT 0 1
```

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













