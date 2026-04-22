# Уровни изоляции транзакций в PostgreSQL 17

## Цели
* Понять идею уровней изоляций транзакций
* Какие уровни с какими аномалиями борятся

## Подготовка окружения
* Используем Docker контейнер из прошлых заданий с PostgreSQL 17
* Подключиться через портейнер к контейнеру, открыть две сессии и отключить в них автокоммит
* Создать таблицу orders и вставить несколько строк

## 1. Подключаемся, создаем таблицу, открываем 2 сессии

<img width="1773" height="664" alt="image" src="https://github.com/user-attachments/assets/412616a7-bde6-4e4e-8646-74f7b6a5711a" />

```sql
postgres=# CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT now(),
    amount NUMERIC
);

INSERT INTO orders (amount) VALUES (100), (200);
CREATE TABLE
INSERT 0 2
postgres=*# COMMIT;
COMMIT
```
Проверяем что данные есть и видны в обоих сессиях

<img width="1565" height="248" alt="image" src="https://github.com/user-attachments/assets/c2979a43-425e-48ab-a4bb-de46ebd62cfe" />

## 2. Сценарий 1: Уровень изоляции READ COMMITTED
## Сессия 2
Начинаем транзакцию, читаем данные, считаем сумму

```sql
postgres=# BEGIN;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

SELECT COUNT(*) AS order_count, SUM(amount) AS total_sum
FROM orders;
BEGIN
SET
 order_count | total_sum 
-------------+-----------
           2 |       300
```

## Сессия 1 (параллельно, пока сессия 2 не закоммитила)

```sql
postgres=# BEGIN;
INSERT INTO orders (amount) VALUES (300);
COMMIT;
BEGIN
INSERT 0 1
```
## Сессия 2 (продолжение)
```sql
postgres=*# SELECT COUNT(*) AS order_count, SUM(amount) AS total_sum
FROM orders;

COMMIT;
 order_count | total_sum 
-------------+-----------
           1 |       600
(1 row)
```
## 2.1 Результат READ COMMITTED
* Первый SELECT в сессии 2: count=2, sum=300 (данные до вставки).
* Второй SELECT в сессии 2: count=3, sum=600 (видит новый заказ из сессии 1, т.к. тот уже закоммичен).
* После COMMIT в сессии 2 — данные сохраняются, но агрегат уже не меняется в рамках завершённой транзакции.
* 
Объяснение поведения.
READ COMMITTED позволяет видеть только закоммиченные изменения других транзакций. При повторном запросе транзакция видит новый закоммиченный заказ.


## 3. Сценарий 2: Уровень изоляции REPEATABLE READ
Повторим все с нуля с новым уровнем изоляции, перезайдем в сессии и выставим autocommit = off, пересоздадим табличку

<img width="1488" height="173" alt="image" src="https://github.com/user-attachments/assets/98972ca5-8d9d-450f-85be-33ab42e57c98" />

```sql
postgres=# CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT now(),
    amount NUMERIC
);

INSERT INTO orders (amount) VALUES (100), (200);
COMMIT;
CREATE TABLE
INSERT 0 2
COMMIT
```

### Сессия 2 
```sql
postgres=# BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN
SET
postgres=*# SELECT COUNT(*), SUM(amount)
FROM orders
postgres-*# ;
 count | sum 
-------+-----
     2 | 300
(1 row)
```

### Сессия 1. Сделаем insert 2 раза, до sum=1100
```sql
postgres=# BEGIN;
INSERT INTO orders (amount) VALUES (400);
COMMIT;
BEGIN
INSERT 0 1
COMMIT
```

### Сессия 2 (продолжение)
```sql
postgres=*# SELECT COUNT(*), SUM(amount)
FROM orders;
 count | sum 
-------+-----
     2 | 300
(1 row)
```

После завершения транзакции сделаем ещё один SELECT во 2 сессии
```sql
postgres=# SELECT COUNT(*), SUM(amount)
FROM orders;
 count | sum  
-------+------
     4 | 1100
(1 row)
```

## 3.1. Результат REPEATABLE READ
* Первый SELECT в сессии 2: count=2, sum=300.
* Второй SELECT в сессии 2: count=2, sum=300 (даже после того, как сессия 1 закоммитила новую строку).
* После COMMIT в сессии 2: новый SELECT покажет count=4 sum=1100.

Объяснение поведения.
REPEATABLE READ гарантирует снапшот данных на момент начала транзакции. Все запросы внутри транзакции видят одни и те же данные, независимо от внешних коммитов.


## 4. Итог

Какой уровень изоляции нужен для отчёта?
Для консистентного отчёта (например, подсчёт заказов за последнюю минуту на момент запуска отчёта) следует использовать REPEATABLE READ.

Причина:
Если отчёт формируется долго или содержит несколько запросов, REPEATABLE READ гарантирует, что все части отчёта увидят одни и те же данные — срез на момент начала формирования. В READ COMMITTED данные могут поменяться между запросами внутри одного отчёта, что приведёт к несогласованности (например, сумма по заказам не будет соответствовать их количеству).
