# Работа с ролями и правами

## Задания
* Создать кластер Postgresql 17, схему и таблицу
* Создать роль и  пользоватаеля
* Настройка прав и анализ поведения при обращении к объектам БД

## Окружение
* Postgresql 17 через Docker контейнер, используем ранее созданный из задания по HW-1

# Docker compose файл

```yaml
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

<img width="1507" height="300" alt="image" src="https://github.com/user-attachments/assets/e7caa440-bf94-4a2d-8a32-bdd2062d98fe" />

Через контейнер подключаемся к Postgresql кластеру. Создаем базу данных, сразу подключаемся к новой БД, создаем схему, таблицу и вставляем строку.

<img width="727" height="419" alt="image" src="https://github.com/user-attachments/assets/9dd79335-66ad-4137-9b1c-ab74a2bf2e01" />

## 1. Создание БД, схемы и таблицы
```sql
root@7ca7365ed9d8:/# psql -U postgres
psql (17.9 (Debian 17.9-1.pgdg13+1))
Type "help" for help.

postgres=# create database testdb
postgres-# ;
CREATE DATABASE

postgres=# \c testdb;
You are now connected to database "testdb" as user "postgres".

postgres=# create schema testnm;
CREATE SCHEMA

testdb=# create table
testdb-# testnm.t1(c1 int);
CREATE TABLE

testdb=# insert into testnm.t1 values(666);
INSERT 0 1

testdb=# select * from testnm.t1;
 c1  
-----
 666
(1 row)
```

## 2. Создание роли readonly и выдача прав
```sql
testdb=# CREATE ROLE readonly;
CREATE ROLE

testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT

testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT

testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
```

## 2. Создание пользователя testread и выдача прав
```sql
testdb=# CREATE USER testread WITH PASSWORD 'test123';
CREATE ROLE
```

Назначаем роль readonly
```sql
GRANT readonly TO testread;
```



## 3. Првоерка прав
Подключаемся к бд testdb под пользователем testread 

```sql
root@7ca7365ed9d8:/# psql -h localhost -U testread -d testdb -W
Password: 
psql (17.9 (Debian 17.9-1.pgdg13+1))
Type "help" for help.
```
и делаем select к таблице
```sql
testdb=> select * from testnm.t1;
 c1  
-----
 666
(1 row)
```

## Все предсказуемо, так как права у роли есть и эта роль назначена пользователю, а значит и права есть у пользователя.

## Пробуем под testread создать или вставить данные в существующую таблицу testnm.t1

```sql
create table testnm.t2(c2 int);
ERROR:  permission denied for schema testnm

insert into testnm.t1
testdb-> values(555);
ERROR:  permission denied for table t1
```

Поведение верное, прав на эти команды у пользователя нет.

4. ## Меняем дефолтное поведение прав на объекты

 ```sql
postgres=# ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES TO readonly;
ALTER DEFAULT PRIVILEGES
```

Создаем новый объект в схеме testnm и вставим данные
 ```sql
postgres=# create table testnm.t3(c3 integer);
CREATE TABLE

postgres=# insert into testnm.t3
values (555);
INSERT 0 1
```

5. Снова зайдем под пользователем testread и сделаем select к новой таблице
 ```sql
root@7ca7365ed9d8:/# psql -h localhost -U testread -d testdb -W
Password: 
psql (17.9 (Debian 17.9-1.pgdg13+1))
Type "help" for help.

testdb=> select * from testnm.t3
;
 c3  
-----
 555
(1 row)
```

Права есть, хотя явно мы их не выдавали. Команда
 ```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES TO readonly
```
как бы говорит, что на все объекты который будут созданы в схеме testnm, дай правай на select.


# Возможное неожиданное/неявное поведение.

В Postgresql имеется механика дефолтного создания объектов в определенной схеме, путь этот можно увидет через команду SHOW search_path;
В нашем случае эта команда показывает
```sql
testdb=> SHOW search_path;
   search_path   
-----------------
 "$user", public
```

Если бы при создании таблицы я не указывал явно схему testnm, таблица t1 создалась бы в схеме public, что могло бы запутать.
Поэтому при создании/изменении каких либо объектов в Postgresql лучше не лениться и указывать полный путь, а именно схема.имя_объекта.



