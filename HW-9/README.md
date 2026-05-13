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
