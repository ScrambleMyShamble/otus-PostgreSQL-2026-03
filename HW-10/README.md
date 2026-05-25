# Виды индексов. Работа с индексами и оптимизация запросов

## Цель
* Уметь создавать индексные поля
* пользоваться командой EXPLAIN
* редактировать, обновлять и удалять индексы

## Окружение
Docker контейнер с Postgresql 17

## 1. Подготовка тестовых данных
Создаем тестовую таблицу и заполняем данными
```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200),
    description TEXT,
    category VARCHAR(50),
    price DECIMAL(10, 2),
    stock_quantity INTEGER,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    search_vector tsvector
);
CREATE TABLE
```

Вставляем тестовые данные (10000 строк)
```sql
INSERT INTO products (name, description, category, price, stock_quantity)
SELECT 
    'Product ' || i,
    'Detailed description for product ' || i || ' with some technical specifications and features',
    CASE (i % 5)
        WHEN 0 THEN 'Electronics'
        WHEN 1 THEN 'Clothing'
        WHEN 2 THEN 'Books'
        WHEN 3 THEN 'Food'
        ELSE 'Furniture'
    END,
    (random() * 1000)::numeric(10,2),
    (random() * 100)::int
FROM generate_series(1, 10000) i;
INSERT 0 10000
```

Обновляем search_vector для полнотекстового поиска
```sql
UPDATE products SET 
    search_vector = to_tsvector('russian', name || ' ' || description);
UPDATE 10000
```

## 2. Создать обычный btree индекс и показать EXPLAIN
```sql
CREATE INDEX idx_products_price ON products(price);
CREATE INDEX
```

Анализируем запрос с использованием индекса
```sql
EXPLAIN ANALYZE SELECT * FROM products WHERE price BETWEEN 100 AND 500;
 Bitmap Heap Scan on products  (cost=93.86..712.70 rows=4056 width=238) (actual time=1.520..2.982 rows=4057 loops=1)
   Recheck Cond: ((price >= '100'::numeric) AND (price <= '500'::numeric))
   Heap Blocks: exact=346
   ->  Bitmap Index Scan on idx_products_price  (cost=0.00..92.84 rows=4056 width=0) (actual time=1.263..1.264 rows=4057 loops=1)
         Index Cond: ((price >= '100'::numeric) AND (price <= '500'::numeric))
 Planning Time: 0.182 ms
 Execution Time: 3.311 ms
(7 rows)
```

## 3. Индекс для полнотекстового поиска
```sql
CREATE INDEX idx_products_search ON products USING GIN(search_vector);
CREATE INDEX
```

Запрос с полнотекстовым поиском
```sql
EXPLAIN ANALYZE 
SELECT * FROM products 
WHERE search_vector @@ to_tsquery('russian', 'description & features');

 Bitmap Heap Scan on products  (cost=86.40..769.39 rows=10000 width=238) (actual time=2.741..5.506 rows=10000 loops=1)
   Recheck Cond: (search_vector @@ '''descript'' & ''featur'''::tsquery)
   Heap Blocks: exact=346
   ->  Bitmap Index Scan on idx_products_search  (cost=0.00..83.90 rows=10000 width=0) (actual time=2.654..2.655 rows=10000 loops=1)
         Index Cond: (search_vector @@ '''descript'' & ''featur'''::tsquery)
 Planning Time: 0.418 ms
 Execution Time: 6.410 ms
(7 rows)
```

Показываем использование индекса
```sql
EXPLAIN (ANALYZE, BUFFERS) 
SELECT id, name, description 
FROM products 
WHERE search_vector @@ to_tsquery('russian', 'technical & specifications');

 Bitmap Heap Scan on products  (cost=86.40..769.39 rows=10000 width=101) (actual time=2.037..3.563 rows=10000 loops=1)
   Recheck Cond: (search_vector @@ '''technic'' & ''specif'''::tsquery)
   Heap Blocks: exact=346
   Buffers: shared hit=357
   ->  Bitmap Index Scan on idx_products_search  (cost=0.00..83.90 rows=10000 width=0) (actual time=1.989..1.990 rows=10000 loops=1)
         Index Cond: (search_vector @@ '''technic'' & ''specif'''::tsquery)
         Buffers: shared hit=11
 Planning:
   Buffers: shared hit=1
 Planning Time: 0.225 ms
 Execution Time: 4.081 ms
(11 rows)
```

## 4. Индекс на часть таблицы (частичный индекс)
```sql
CREATE INDEX idx_active_high_price ON products(price) 
WHERE is_active = true AND price > 100;
CREATE INDEX
```

Запрос, использующий частичный индекс
```sql
EXPLAIN ANALYZE 
SELECT * FROM products 
WHERE is_active = true AND price > 100 AND price < 300;

 Bitmap Heap Scan on products  (cost=44.24..633.13 rows=2059 width=238) (actual time=0.477..0.985 rows=2057 loops=1)
   Recheck Cond: ((price < '300'::numeric) AND is_active AND (price > '100'::numeric))
   Heap Blocks: exact=345
   ->  Bitmap Index Scan on idx_active_high_price  (cost=0.00..43.73 rows=2059 width=0) (actual time=0.439..0.440 rows=2057 loops=1)
         Index Cond: (price < '300'::numeric)
 Planning Time: 0.326 ms
 Execution Time: 1.061 ms
(7 rows)
```

Индекс на поле с функцией
```sql
CREATE INDEX idx_lower_name ON products(LOWER(name));
CREATE INDEX
```

Запрос, использующий функциональный индекс
```sql
postgres=# EXPLAIN ANALYZE 
SELECT * FROM products 
WHERE LOWER(name) = 'product 500';
                                                       QUERY PLAN                                                       
------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=4.67..155.19 rows=50 width=238) (actual time=0.115..0.116 rows=1 loops=1)
   Recheck Cond: (lower((name)::text) = 'product 500'::text)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on idx_lower_name  (cost=0.00..4.66 rows=50 width=0) (actual time=0.102..0.102 rows=1 loops=1)
         Index Cond: (lower((name)::text) = 'product 500'::text)
 Planning Time: 2.409 ms
 Execution Time: 0.174 ms
(7 rows)
```

## 5. Составной индекс (на несколько полей)
```sql
CREATE INDEX idx_category_price ON products(category, price);
CREATE INDEX
```

Запрос, использующий составной индекс
```sql
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM products 
WHERE category = 'Electronics' AND price BETWEEN 200 AND 500;

Bitmap Heap Scan on products  (cost=19.92..612.26 rows=599 width=238) (actual time=0.406..0.955 rows=608 loops=1)
   Recheck Cond: (((category)::text = 'Electronics'::text) AND (price >= '200'::numeric) AND (price <= '500'::numeric))
   Heap Blocks: exact=303
   Buffers: shared hit=303 read=5
   ->  Bitmap Index Scan on idx_category_price  (cost=0.00..19.77 rows=599 width=0) (actual time=0.257..0.257 rows=608 loops=1)
         Index Cond: (((category)::text = 'Electronics'::text) AND (price >= '200'::numeric) AND (price <= '500'::numeric))
         Buffers: shared read=5
 Planning:
   Buffers: shared hit=33 read=1 dirtied=3
 Planning Time: 0.839 ms
 Execution Time: 1.026 ms
(11 rows)
```

Составной индекс с сортировкой
```sql
CREATE INDEX idx_category_price_stock ON products(category, price DESC, stock_quantity);
CREATE INDEX
```

Запрос для сложной фильтрации
```sql
EXPLAIN ANALYZE 
SELECT * FROM products 
WHERE category = 'Books' 
  AND price < 50 
  AND stock_quantity > 10 
ORDER BY price DESC;

 Sort  (cost=263.60..263.82 rows=88 width=238) (actual time=0.194..0.198 rows=91 loops=1)
   Sort Key: price DESC
   Sort Method: quicksort  Memory: 48kB
   ->  Bitmap Heap Scan on products  (cost=5.29..260.76 rows=88 width=238) (actual time=0.072..0.167 rows=91 loops=1)
         Recheck Cond: (((category)::text = 'Books'::text) AND (price < '50'::numeric))
         Filter: (stock_quantity > 10)
         Rows Removed by Filter: 8
         Heap Blocks: exact=90
         ->  Bitmap Index Scan on idx_category_price  (cost=0.00..5.27 rows=98 width=0) (actual time=0.055..0.056 rows=99 loops=1)
               Index Cond: (((category)::text = 'Books'::text) AND (price < '50'::numeric))
 Planning Time: 0.170 ms
 Execution Time: 0.219 ms
(12 rows)
```



