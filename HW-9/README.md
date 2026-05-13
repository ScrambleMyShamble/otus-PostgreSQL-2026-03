# Выборка данных, виды join'ов. Применение и оптимизация
## Цели
* Зачем нужны join
* Понимать отличия между ними
* Умение оптимизировать запросы соединения

## Окружение
* Docker контейнера с Postgresql 17


## 1. Подготовка. Подключимся к контейнеру, создадим и заполним таблицы

<img width="771" height="432" alt="image" src="https://github.com/user-attachments/assets/387e4635-c347-491c-90ff-37fd85aaaa0b" />


Будем использовать 3 таблицы: клиенты, заказы, позиций заказов
``` sql
CREATE TABLE customers (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

CREATE TABLE order_items (
    id INT PRIMARY KEY,
    order_id INT,
    product_name VARCHAR(100),
    quantity INT,
    FOREIGN KEY (order_id) REFERENCES orders(id)
);

CREATE TABLE
CREATE TABLE
CREATE TABLE
```

И заполним данными
``` sql
postgres=# INSERT INTO customers VALUES (1, 'Иван'), (2, 'Мария'), (3, 'Петр');
INSERT INTO orders VALUES (101, 1, '2024-01-10'), (102, 2, '2024-01-15'), (103, NULL, '2024-01-20');
INSERT INTO order_items VALUES (1001, 101, 'Ноутбук', 1), (1002, 102, 'Мышь', 2), (1003, 999, 'Клавиатура', 1);
INSERT 0 3
INSERT 0 3
INSERT 0 3
```


## 2. Прямое соединение (INNER JOIN)
``` sql
SELECT c.name, o.order_date, oi.product_name, oi.quantity
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
INNER JOIN order_items oi ON o.id = oi.order_id;

 name  | order_date | product_name | quantity 
-------+------------+--------------+----------
 Иван  | 2024-01-10 | Ноутбук      |        1
 Мария | 2024-01-15 | Мышь         |        2
(2 rows)
```
Возвращает только те строки, где есть совпадение во всех трёх таблицах.
Если у заказа нет клиента (customer_id IS NULL) или позиций, такие заказы исключаются.

## 3. Левостороннее соединение (LEFT JOIN)
``` sql
postgres=# SELECT o.id, o.order_date, c.name
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id;
 id  | order_date | name  
-----+------------+-------
 101 | 2024-01-10 | Иван
 102 | 2024-01-15 | Мария
 103 | 2024-01-20 | 
(3 rows)
```
Берутся все строки из левой таблицы (orders), а из правой (customers) — только совпадающие.
Если клиента нет — в колонках c.name будет NULL.

## 4. Кросс-соединение (CROSS JOIN)
``` sql
postgres=# SELECT c.name, o.order_date
FROM customers c
CROSS JOIN orders o;
 name  | order_date 
-------+------------
 Иван  | 2024-01-10
 Мария | 2024-01-10
 Петр  | 2024-01-10
 Иван  | 2024-01-15
 Мария | 2024-01-15
 Петр  | 2024-01-15
 Иван  | 2024-01-20
 Мария | 2024-01-20
 Петр  | 2024-01-20
(9 rows)
```
Каждая строка из customers соединяется с каждой строкой из orders.
Количество строк = (число клиентов) × (число заказов).


## 5. Полное соединение (FULL OUTER JOIN)
``` sql
postgres=# SELECT c.name, o.id AS order_id, o.order_date
FROM customers c
FULL OUTER JOIN orders o ON c.id = o.customer_id;
 name  | order_id | order_date 
-------+----------+------------
 Иван  |      101 | 2024-01-10
 Мария |      102 | 2024-01-15
       |      103 | 2024-01-20
 Петр  |          | 
(4 rows)
```

Объединяет результаты LEFT JOIN и RIGHT JOIN.
Все строки из обеих таблиц сохраняются, недостающие значения заполняются NULL.

## 6. Запрос с разными типами соединений
``` sql
SELECT 
    c.name AS customer_name,
    o.id AS order_id,
    o.order_date,
    oi.product_name,
    oi.quantity
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id   -- только клиенты с заказами
LEFT JOIN order_items oi ON o.id = oi.order_id; -- все позиции этих заказов (если нет позиций -> NULL)
 customer_name | order_id | order_date | product_name | quantity 
---------------+----------+------------+--------------+----------
 Иван          |      101 | 2024-01-10 | Ноутбук      |        1
 Мария         |      102 | 2024-01-15 | Мышь         |        2
(2 rows)
```
Сначала INNER JOIN фильтрует клиентов, у которых есть заказы.
Затем LEFT JOIN присоединяет позиции заказов, сохраняя даже те заказы, у которых нет позиций (в product_name и quantity будет NULL).
Полезно, когда нужно показать всех заказчиков с заказами, но список товаров может отсутствовать.








