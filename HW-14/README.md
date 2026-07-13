# Хранимые функции и процедуры. Триггеры.

## Цель
* Научиться работать и разбирать событийные триггеры

## 1. Подготовка
``` sql
demo=# CREATE SCHEMA pract_functions;
CREATE SCHEMA
demo=# SET search_path = pract_functions, public;
SET
demo=# CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
CREATE TABLE
demo=# INSERT INTO goods (goods_id, good_name, good_price)
VALUES  (1, 'Спички хозайственные', .50),
                (2, 'Автомобиль Ferrari FXX K', 185000000.01);
INSERT 0 2
demo=# CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);
CREATE TABLE
demo=# INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
INSERT 0 4
demo=# SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
        good_name         |     sum      
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)

demo=# CREATE TABLE good_sum_mart
(
        good_name   varchar(63) NOT NULL,
        sum_sale        numeric(16, 2)NOT NULL
);
CREATE TABLE
```

## 2. СОЗДАНИЕ ТРИГГЕРА ДЛЯ ПОДДЕРЖКИ ВИТРИНЫ

Для начала создаем триггерную функцию
``` sql
CREATE OR REPLACE FUNCTION pract_functions.update_good_sum_mart()
RETURNS TRIGGER AS
$$
DECLARE
    v_good_name pract_functions.goods.good_name%TYPE;
    v_good_price pract_functions.goods.good_price%TYPE;
    v_diff numeric(16, 2);
BEGIN
    -- Получаем данные товара с указанием схемы
    SELECT good_name, good_price INTO v_good_name, v_good_price
    FROM pract_functions.goods
    WHERE goods_id = COALESCE(NEW.good_id, OLD.good_id);
    
    -- Вычисляем разницу в зависимости от операции
    IF TG_OP = 'DELETE' THEN
        v_diff := -(OLD.sales_qty * v_good_price);
    ELSIF TG_OP = 'INSERT' THEN
        v_diff := NEW.sales_qty * v_good_price;
    ELSE -- UPDATE
        v_diff := (NEW.sales_qty - OLD.sales_qty) * v_good_price;
    END IF;
    
    -- Обновляем или вставляем запись в витрину
    INSERT INTO pract_functions.good_sum_mart (good_name, sum_sale)
    VALUES (v_good_name, v_diff)
    ON CONFLICT (good_name) DO UPDATE
    SET sum_sale = pract_functions.good_sum_mart.sum_sale + EXCLUDED.sum_sale;
    
    RETURN COALESCE(NEW, OLD);
END;
$$
LANGUAGE plpgsql;
```

TG_OP - это специальная переменная в триггерных функциях PostgreSQL, которая содержит тип операции, вызвавшей триггер.

Создаем сам триггер, указав функцию и таблицу
``` sql
CREATE TRIGGER trg_update_good_sum_mart
AFTER INSERT OR UPDATE OR DELETE ON pract_functions.sales
FOR EACH ROW
EXECUTE FUNCTION pract_functions.update_good_sum_mart();
```


## 3. ТЕСТИРОВАНИЕ РАБОТЫ ТРИГГЕРА
Проверим начальное состояние витрины
``` sql
SELECT * FROM pract_functions.good_sum_mart ORDER BY good_name;
```
Пустой ответ

Тест INSERT: добавляем продажу спичек (1 шт.)
``` sql
INSERT INTO sales (good_id, sales_qty) VALUES (1, 5);
demo=# SELECT * FROM good_sum_mart ORDER BY good_name;
      good_name       | sum_sale 
----------------------+----------
 Спички хозайственные |     2.50
(1 row)
```

Тест UPDATE: изменяем количество в продаже (было 5, стало 3)
``` sql
UPDATE sales SET sales_qty = 3 WHERE sales_id = (SELECT MAX(sales_id) FROM sales);
UPDATE 1
```
Проверяем витрину
``` sql
demo=# SELECT * FROM good_sum_mart ORDER BY good_name;
      good_name       | sum_sale 
----------------------+----------
 Спички хозайственные |     1.50
(1 row)
```

Тест DELETE: удаляем последнюю продажу
``` sql
DELETE FROM sales WHERE sales_id = (SELECT MAX(sales_id) FROM sales);
DELETE 1
```
Проверяем витрину
``` sql
demo=# SELECT * FROM good_sum_mart ORDER BY good_name;
      good_name       | sum_sale 
----------------------+----------
 Спички хозайственные |     0.00
(1 row)
```

Тест: добавляем продажу Ferrari (1 шт.)
``` sql
INSERT INTO sales (good_id, sales_qty) VALUES (2, 1);
INSERT 0 1
```
Проверяем витрину
``` sql
SELECT * FROM good_sum_mart ORDER BY good_name;
        good_name         |   sum_sale   
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |         0.00
(2 rows)
```

Сравниваем с отчетом (должны совпадать)
``` sql
demo=# SELECT G.good_name, COALESCE(sum(G.good_price * S.sales_qty), 0) as sum_sale
FROM goods G
LEFT JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name
ORDER BY G.good_name;
        good_name         |   sum_sale   
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        0.00
(2 rows)
```






















