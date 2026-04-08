# Установка PostgreSQL 18 внутри контейнера Docker и его переустановка.

## Описание задания
Создать и запустить контейнер с PostgreSQL 18, вынести каталог данных базы на хост, убедиться после пересоздания контйнера что все данные на месте.

## Окружение
* Vmvare с ОС Ubuntu
* Docker/portainer
* PostgreSQL: образ postgres:17
* Пользователь БД: postgres
* База данных: otus
* Каталог с данными: ./var/lib/postgresql

## Docker Compose
Файл docker-compose.yml
```yaml
version: '3.3'

version: '3.3'

services:
  postgres:
    image: postgres:17
    container_name: postgres-db
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "5432:5432"
    networks:
      - zabbix-net
      - monitoring-net
    volumes:
      - /var/lib/postgresql/data:/var/lib/postgresql/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 30s
      timeout: 10s
      retries: 3
networks:
  zabbix-net:
    external: true
  monitoring-net:
    external: true

    
#volumes:
#  postgres_data:
#    driver: local
```

В стэк файле в пункте volume указываем диск хоста
```dockerfile
  volumes:
      - /var/lib/postgresql/data:/var/lib/postgresql/data
```

## 1. Запуск контейнера
docker compose up -d
Проверяем запустился ли
CONTAINER ID   IMAGE              COMMAND                  CREATED          STATUS                       PORTS                                     NAMES
e6d66d89ac5d   postgres:17   "docker-entrypoint.s…"   14 minutes ago   Up 14 minutes (healthy)      0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp    postgres-db

## 2. Подключение к контейнеру через Portainer и создание базы данных otus и таблицы orders_test
```sql
create database otus;

create table orders_test (id integer, orders text);

insert into orders_test(id, orders)
values (1, 'Костюм слона в натуральную величину'),
(2, 'Банка сгущенки');
```

## 3. Подключиться с хоста через dbeaver к базе данных otus на виртуальной машине и сделать проверить данные
```sql
select * from orders_test;
<img width="477" height="116" alt="image" src="https://github.com/user-attachments/assets/1d313774-d1ed-40db-bbeb-70bc17b21f45" />
```

## 4. Удаляем stack, а следовательно и контейнер с postgresql и проверяем что базы больше нет
<img width="399" height="106" alt="image" src="https://github.com/user-attachments/assets/10f31878-bf4c-43b0-ab9a-c1abfa23ae11" />
<img width="642" height="309" alt="image" src="https://github.com/user-attachments/assets/9433d466-cd8f-41b7-be0a-52cd645ec621" />

## 5. Создаем postgresql контейнер и запускаем его, делаем select к базе
<img width="442" height="333" alt="image" src="https://github.com/user-attachments/assets/2da0aa28-0a7a-4a8f-8c8e-3f2d68cf4b15" />

# Итог
При монтировании хранилища контейнера в диск хоста, после удаления самого контейнера все данные остаются в сохранности, так каходятся на внешнем для контейнера диске. После установки Postgresql мы можем смонтировать диск снова и получить те же данные.



