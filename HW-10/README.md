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






    
