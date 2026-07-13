# Секционирование
## Цель
* создавать секционированные таблицы
* обслуживать секционированные таблицы
* увеличить производительность запросов

### 1.Выбор таблицы для секционирования и определение типа секционирования
Как и сказано в задаче, выбираем таблицу bookings
Причины:
* Естественный временной диапазон: Поле book_date (дата бронирования) идеально подходит для секционирования по диапазону
* Большой объем данных: В реальных авиакомпаниях таблица бронирований быстро растет и становится одной из самых больших.
* Часто выполняются запросы по датам
* Секционирование позволяет легко архивировать старые данные
* Ключевая таблица: bookings является центральной таблицей, от которой зависят другие

1.2 Тип секционирования
Выбранный тип: секционирование по диапазону (RANGE) по полю book_date
Обоснование:
* Данные имеют естественную временную шкалу
* Запросы часто фильтруются по датам
* Можно архивировать старые данные
* Равномерное распределение данных по времени

### 2. Создание секционированной таблицы
``` sql
CREATE TABLE bookings_partitioned (
    book_ref character(6) NOT NULL,
    book_date timestamp with time zone NOT NULL,
    total_amount numeric(10,2) NOT NULL
) PARTITION BY RANGE (book_date);

CREATE TABLE
```

### 2.1 Создаем секции по месяцам (2025 год)
``` sql
CREATE TABLE bookings_2025_01 PARTITION OF bookings_partitioned
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE bookings_2025_02 PARTITION OF bookings_partitioned
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

CREATE TABLE bookings_2025_03 PARTITION OF bookings_partitioned
    FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');

CREATE TABLE bookings_2025_04 PARTITION OF bookings_partitioned
    FOR VALUES FROM ('2025-04-01') TO ('2025-05-01');

CREATE TABLE bookings_2025_05 PARTITION OF bookings_partitioned
    FOR VALUES FROM ('2025-05-01') TO ('2025-06-01');

CREATE TABLE bookings_2025_06 PARTITION OF bookings_partitioned
    FOR VALUES FROM ('2025-06-01') TO ('2025-07-01');

CREATE TABLE bookings_2025_07 PARTITION OF bookings_partitioned
    FOR VALUES FROM ('2025-07-01') TO ('2025-08-01');

CREATE TABLE bookings_2025_08 PARTITION OF bookings_partitioned
    FOR VALUES FROM ('2025-08-01') TO ('2025-09-01');

CREATE TABLE bookings_2025_09 PARTITION OF bookings_partitioned
    FOR VALUES FROM ('2025-09-01') TO ('2025-10-01');

CREATE TABLE bookings_2025_10 PARTITION OF bookings_partitioned
    FOR VALUES FROM ('2025-10-01') TO ('2025-11-01');

CREATE TABLE bookings_2025_11 PARTITION OF bookings_partitioned
    FOR VALUES FROM ('2025-11-01') TO ('2025-12-01');

CREATE TABLE bookings_2025_12 PARTITION OF bookings_partitioned
    FOR VALUES FROM ('2025-12-01') TO ('2026-01-01');
CREATE TABLE
```

### 3. Миграция данных

Проверяем существующие данные
``` sql
SELECT COUNT(*) FROM bookings.bookings;
1292893
```

### 3.1 Создаем таблицу для мониторинга миграции
``` sql
CREATE TABLE migration_log (
    migration_id SERIAL PRIMARY KEY,
    table_name TEXT,
    records_moved INTEGER,
    migration_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE
```

Переносим данные из старой таблицы в новую
``` sql
INSERT INTO bookings_partitioned (book_ref, book_date, total_amount)
SELECT book_ref, book_date, total_amount
FROM bookings.bookings
WHERE book_date >= '2025-01-01' AND book_date < '2026-01-01';
Updated Rows 1292893
```
Запись в лог
``` sql
INSERT INTO migration_log (table_name, records_moved) 
VALUES ('bookings', (SELECT COUNT(*) FROM bookings.bookings));
```

Проверяем распределение данных по секциям
``` sql
SELECT 
    tableoid::regclass AS partition,
    COUNT(*) AS row_count,
    MIN(book_date) AS min_date,
    MAX(book_date) AS max_date
FROM bookings_partitioned
GROUP BY tableoid
ORDER BY partition;

bookings_2025_09	446319	2025-09-01 03:00:06.265 +0300	2025-09-30 23:59:53.923 +0300
bookings_2025_10	434287	2025-10-01 00:00:03.529 +0300	2025-10-31 23:59:48.434 +0300
bookings_2025_11	410680	2025-11-01 00:00:01.030 +0300	2025-11-30 23:59:59.941 +0300
bookings_2025_12	1607	2025-12-01 00:00:23.471 +0300	2025-12-01 02:59:28.616 +0300
```

### 4.Оптимизация запросов и тестирование производительности
Запрос до секционирования через таблицу bookings_before 
``` sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) 
SELECT book_ref, book_date, total_amount 
FROM bookings_before 
WHERE book_date BETWEEN '2025-09-01' AND '2025-09-30';

Seq Scan on bookings.bookings_before  (cost=0.00..27649.40 rows=430323 width=21) (actual time=0.018..55.926 rows=431782 loops=1)
  Output: book_ref, book_date, total_amount
  Filter: ((bookings_before.book_date >= '2025-09-01 00:00:00+03'::timestamp with time zone) AND (bookings_before.book_date <= '2025-09-30 00:00:00+03'::timestamp with time zone))
  Rows Removed by Filter: 861111
  Buffers: shared hit=2962 read=5294
Planning:
  Buffers: shared hit=5 dirtied=1
Planning Time: 0.220 ms
Execution Time: 67.789 ms
```

После
``` sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) 
SELECT book_ref, book_date, total_amount 
FROM bookings_partitioned 
WHERE book_date >= '2025-09-01 00:00:00+03' 
  AND book_date < '2025-10-01 00:00:00+03';

Seq Scan on bookings.bookings_2025_09 bookings_partitioned  (cost=0.00..19075.57 rows=892459 width=21) (actual time=0.055..58.567 rows=892638 loops=1)
  Output: bookings_partitioned.book_ref, bookings_partitioned.book_date, bookings_partitioned.total_amount
  Filter: ((bookings_partitioned.book_date >= '2025-09-01 00:00:00+03'::timestamp with time zone) AND (bookings_partitioned.book_date < '2025-10-01 00:00:00+03'::timestamp with time zone))
  Buffers: shared hit=3003 read=2683
Planning:
  Buffers: shared hit=6
Planning Time: 0.143 ms
Execution Time: 83.646 ms
```

### 4. Дополнительные оптимизации
Создаем индексы для каждой секции
``` sql
CREATE INDEX idx_bookings_2025_01_date ON bookings_2025_01(book_date);
```
И так далее для каждой секции

Используем частичный индекс для частых запросов
``` sql
CREATE INDEX idx_bookings_high_value ON bookings_partitioned(total_amount) 
    WHERE total_amount > 10000;
```

### 5. Проверка, что операции вставки, обновления и удаления работают корректно.
Вставка
``` sql
INSERT INTO bookings_partitioned (book_ref, book_date, total_amount)
VALUES ('AB1234', '2025-09-15 12:00:00+03', 5000.00);
Updated Rows 1
```

``` sql
INSERT INTO bookings_partitioned (book_ref, book_date, total_amount)
VALUES ('CD5678', '2025-12-15 12:00:00+03', 7500.00);
Updated Rows 1
```

``` sql
SELECT * FROM bookings_partitioned 
WHERE book_ref IN ('AB1234', 'CD5678');

AB1234	2025-09-15 12:00:00.000 +0300	5000.00
CD5678	2025-12-15 12:00:00.000 +0300	7500.00
```

Обновление
``` sql
UPDATE bookings_partitioned 
SET total_amount = 6000.00 
WHERE book_ref = 'AB1234';
Updated Rows 1
```

``` sql
SELECT * FROM bookings_partitioned WHERE book_ref = 'AB1234';
AB1234	2025-09-15 12:00:00.000 +0300	6000.00
```
Удаление
``` sql
DELETE FROM bookings_partitioned 
WHERE book_ref = 'CD5678';
DELETE 1
```

Проверка
``` sql
SELECT * FROM bookings_partitioned WHERE book_ref = 'CD5678';

 book_ref | book_date | total_amount 
----------+-----------+--------------
(0 rows)
```

ПРОВЕРКА РАСПРЕДЕЛЕНИЯ ПОСЛЕ DML
``` sql
SELECT 
    tableoid::regclass AS partition_name,
    COUNT(*) AS row_count
FROM bookings_partitioned
GROUP BY tableoid
ORDER BY partition_name;

bookings_2025_09	892639
bookings_2025_10	868574
bookings_2025_11	821360
bookings_2025_12	3215
```




