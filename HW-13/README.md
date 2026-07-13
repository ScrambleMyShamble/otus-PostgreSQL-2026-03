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

```










































