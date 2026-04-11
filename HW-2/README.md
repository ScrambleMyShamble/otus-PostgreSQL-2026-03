# PostgreSQL 17 перенести данные на внешний диск

## Описание
  * Создать внешний диск и подключить его к вирткуальной машине
  * Перенести данные Postgresql на внешний диск
  * Сразу настроить Postgresql на работу с внешним диском

## Релизация через Docker
  * Внутри стэка 2 сервиса для ос Ubuntu
  * ubuntu-vm1 и ubuntu-vm2 из которых получаем 2 контейнера эмулирующие ВМ
  * Внешний диск для подключения

## Dcoker файл

```yaml
services:
  ubuntu-vm1:
    image: ubuntu:24.04
    container_name: vm1
    hostname: vm1
    stdin_open: true
    tty: true
    volumes:
      - postgres-disk:/mnt/external-disk
    command: ["/bin/bash"]

  ubuntu-vm2:
    image: ubuntu:24.04
    container_name: vm2
    hostname: vm2
    stdin_open: true
    tty: true
    volumes:
      - postgres-disk:/mnt/external-disk
    command: ["/bin/bash"]

volumes:
  postgres-disk:
    driver: local
```

![photo_2026-04-11_16-14-40](https://github.com/user-attachments/assets/1e07c985-1eb9-438b-8dad-19a6535d0668)


## 1. Устновка/настройка
Через портейнер проваливаемся внутрь контейнера и выполняем следующие команды:

![photo_2026-04-11_16-14-46](https://github.com/user-attachments/assets/4d7f5741-3369-404d-a58d-c5d95c299530)


```bash
apt update
apt install -y curl gnupg2 lsb-release
sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg
apt update
apt install -y postgresql-17 postgresql-client-17
service postgresql start
```
Проверяем что все запустилось

```bash
pg_lsclusters
```

```bash
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```

Подключаемся к кластеру Postgresql, создаем таблицу и заполняем данными

```bash
psql -U postgres
```

```sql
insert into vm1_test
values
('Data from vm_1 postgres databse');

select * from vm1_test;
              field              
---------------------------------
 Data from vm_1 postgres databse
(1 row)
```

Останавливаем кластер и проверяем

```bash
pg_ctlcluster 17 main stop
```

```bash
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 down   postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```

## 2. Работа с дисками

В стэке указано /mnt/external-disk, это эмуляция внешнего диска для нашего теста.

Создаем директорию для данных Postgresql и выдаем права пользователю postgres

```bash
mkdir -p /mnt/external-disk/pgdata
chown -R postgres:postgres /mnt/external-disk/pgdata
```

Внешний каталог готов для переноса данных и сразу првоеряем упешность операции

```bash
mv /var/lib/postgresql/17 /mnt/external-disk/pgdata/
```

```bash
 ls -la /mnt/external-disk/pgdata/17/main/
```

Видим перенесенные файлы
```bash
total 88
drwx------ 19 postgres postgres 4096 Apr 11 15:11 .
drwxr-xr-x  3 postgres postgres 4096 Apr 11 14:08 ..
-rw-------  1 postgres postgres    3 Apr 11 14:08 PG_VERSION
drwx------  5 postgres postgres 4096 Apr 11 14:08 base
drwx------  2 postgres postgres 4096 Apr 11 14:23 global
drwx------  2 postgres postgres 4096 Apr 11 14:08 pg_commit_ts
drwx------  2 postgres postgres 4096 Apr 11 14:08 pg_dynshmem
drwx------  4 postgres postgres 4096 Apr 11 15:11 pg_logical
drwx------  4 postgres postgres 4096 Apr 11 14:08 pg_multixact
drwx------  2 postgres postgres 4096 Apr 11 14:08 pg_notify
drwx------  2 postgres postgres 4096 Apr 11 14:08 pg_replslot
drwx------  2 postgres postgres 4096 Apr 11 14:08 pg_serial
drwx------  2 postgres postgres 4096 Apr 11 14:08 pg_snapshots
drwx------  2 postgres postgres 4096 Apr 11 15:11 pg_stat
drwx------  2 postgres postgres 4096 Apr 11 14:08 pg_stat_tmp
drwx------  2 postgres postgres 4096 Apr 11 14:08 pg_subtrans
drwx------  2 postgres postgres 4096 Apr 11 14:08 pg_tblspc
drwx------  2 postgres postgres 4096 Apr 11 14:08 pg_twophase
drwx------  4 postgres postgres 4096 Apr 11 14:08 pg_wal
drwx------  2 postgres postgres 4096 Apr 11 14:08 pg_xact
-rw-------  1 postgres postgres   88 Apr 11 14:08 postgresql.auto.conf
-rw-------  1 postgres postgres  130 Apr 11 14:09 postmaster.opts
```

Мы заменили директорию с данными Postgresql, нужно указать это в файле конфигцрации, иначе кластер не поднимется, так как Postgresql, будет искать данные в старом месте.

![photo_2026-04-11_16-14-50](https://github.com/user-attachments/assets/9317c8c1-5f25-4815-ac89-220e51a9767f)

Запускаем и проверяем что все работает

```bash
pg_lsclusters

Ver Cluster Port Status Owner    Data directory                    Log file
17  main    5432 online postgres /mnt/external-disk/pgdata/17/main /var/log/postgresql/postgresql-17-main.log
```

Видим изменившуюся директорию с данными /mnt/external-disk/pgdata/17/main

```sql
SHOW data_directory;
data_directory
 /mnt/external-disk/pgdata/17/main
(1 row)  
```

После всех манипуляций с данными и переносом диска подключаемся к postgres и проверяем что перенесенные данные доступны с "внешнего диска"

```bash
psql -U postgres
psql (17.9 (Ubuntu 17.9-1.pgdg24.04+1))
Type "help" for help.
```

```sql
select * from vm1_test;
              field              
---------------------------------
 Data from vm_1 postgres databse
(1 row)
```



## Задание со звездочкой

Заходим во второй чистый контейнер

![photo_2026-04-11_16-23-59](https://github.com/user-attachments/assets/44210934-c588-455a-8016-05fa7c4f8b3e)

Повторяем пункты с установкой Postgres из первой части, поднимаем кластер и проверяем

```bash
root@vm2:/# service postgresql start
 * Starting PostgreSQL 17 database server                                                                                     [ OK ] 
root@vm2:/# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```

Так как данные мы хотим использовать с "внешнего диска" то грохаем дефолтную директорию кластера Postgresql

```bash
root@vm2:/# rm -rf /var/lib/postgresql/17/main
root@vm2:/# ls -la /var/lib/postgresql/17/
total 8
drwxr-xr-x 2 postgres postgres 4096 Apr 11 15:37 .
drwxr-xr-x 3 postgres postgres 4096 Apr 11 15:34 ..
```

Также заменяем директорию с данными на "внешнюю" в файле конфигурации, по аналогии с первой частью

![photo_2026-04-11_16-24-04](https://github.com/user-attachments/assets/ab2887c4-00a0-4a3e-8d34-b15a4a846934)

<img width="573" height="236" alt="image" src="https://github.com/user-attachments/assets/e2de1b54-aa4e-4ea0-9e07-1d561a456a9f" />


Запускаем кластер и првоеряем что внешний диск подключен, а значит должны получить данные из таблицы vm1_test

```bash
root@vm2:/etc/postgresql/17/main# psql -U postgres
psql (17.9 (Ubuntu 17.9-1.pgdg24.04+1))
Type "help" for help.
```

```sql
postgres=# select *
postgres-# from vm1_test;
              field              
---------------------------------
 Data from vm_1 postgres databse
(1 row)
```


## Подытожим
* Запустили 2 контейнера эмулируя ВМ с ос Ubuntu
* Установили Postgresql на ВМ 1, создали таблицу с данными
* Переместии данные на внешний диск и получили их
* Запустили ВМ №2, удалили дефолтную директорию с данными и прописали внешнюю
* Убедились в правильности работы получив данные с ранее созданного внешнего источника на ВМ 2
