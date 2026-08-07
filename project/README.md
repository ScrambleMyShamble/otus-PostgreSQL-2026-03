## Курс: OTUS – PostgreSQL для администраторов баз данных и разработчиков
### Проектная работа по теме:
### Создание и тестирование высоконагруженного отказоустойчивого кластера PostgreSQL на базе Patroni

### Цели и задачи:
## 1. Цель - Разработать и развернуть отказоустойчивый кластер PostgreSQL 17

## 2. Задачи:
1. Спроектировать архитектуру кластера
2. Создать и настроить 3 виртуальные машины на ОС Ubuntu
3. Развернуть кластер etcd (3 узла)
4. Установить и подготовить PostgreSQL для дальнейшей настройки Patroni
5. Настроить Patroni для управления PostgreSQL (автоматический failover, репликация, восстановление).
6. Настроить балансировку нагрузки через HAProxy
7. Настроить Keepalived
8. Провести функциональное тестирование кластера (создание БД, репликация, failover).

## 3. Используемые технологии
| Элемент | Версия | Функционал |
| --- | ------- | ----- |
| PostgreSQL | 17.10 | СУБД |
| Patroni | 4.1.3 | Управление кластером PostgreSQL |
| Etcd | 3.5.16 | Распределённое хранилище конфигураций |
| ОС + ВМ | Ubuntu 24.04.3 LTS + VMVare| Операционная система + ВМ |
| HAProxy | 2.2.8 | Балансировка нагрузки |
| Keepalived | 2.8 | Отказоустойчивый VIP для HAProxy |

## 4. Архитектура кластера
![cluster](схема.png)

4.1 Компоненты
| Элемент | Адрес | Функционал |
| --- | ------- | ----- |
| pg-master | 192.168.244.131 | Мастер PostgreSQL |
| pg-replica1 | 192.168.244.132 | Реплика PostgreSQL |
| pg-replica2 | 192.168.244.133 | Реплика PostgreSQL |
| etcd | 131/132/133 | 3 узла etcd |
| Keepalived | 192.168.244.140 | Плавающий ip адрес, живет на одной ноде |
| HAProxy | 131/132/133  | Балансировщик |
| Docker + Portainer | 192.168.244.131 | GUI для элементов |

## 5. Настройка linux машин
Единственная проблема, с которой столкнулся при настройке кластера, это меняющийся ip у виртуальных машин, что не очень удобно, поэтому делаем адреса статическими.
Проделываем на каждой ВМ.
![ip](stat_ip.png)

Больше ничего настраивать не нужно, в этом и заключается удобство виртуальных машин, все уже есть из коробки.

## 6. Настройка etcd

В общем случае etcd кластер это:
Создать пользователя etcd(1) -> Создать папки и дать права(2) -> Скачать и распаковать бинарники(3)  -> Написать конфиг ноды(4) -> Написать сервисный файл(5) -> Запустить кластер(6)

6.1 Создаем пользователя 
```bash
sudo useradd -r -s /usr/sbin/nologin -M etcd
```
6.2. создаем директории 
```bash
sudo mkdir -p /opt/etcd/3.5.16/{bin,data,logs,scripts}
```
6.3. создаем переменную окружения
```bash
ETCD_VER="v3.5.16"
```
6.4. скачиваем бинарники
```bash
wget https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
```
6.5. устанавливаем
```bash
tar xzvf etcd-${ETCD_VER}-linux-amd64.tar.gz
```
6.6. перемещаем в нужную директорию
```bash
sudo cp etcd-${ETCD_VER}-linux-amd64/etcd* /opt/etcd/3.5.16/bin/
```
6.7. выдаем права и на запуск
```bash
sudo chmod +x /opt/etcd/3.5.16/bin/*
sudo chown -R root:root /opt/etcd/3.5.16/bin
sudo chown -R etcd:etcd /opt/etcd/3.5.16/data
sudo chown -R etcd:etcd /opt/etcd/3.5.16/logs
```
6.8. бинарники больше не нужны - удаляем
```bash
rm -rf etcd-${ETCD_VER}-linux-amd64 etcd-${ETCD_VER}-linux-amd64.tar.gz
```
6.9. настройка симлинков
```bash
sudo ln -sf /opt/etcd/3.5.16/bin/etcd /usr/local/bin/etcd
sudo ln -sf /opt/etcd/3.5.16/bin/etcdctl /usr/local/bin/etcdctl
```
6.10. Настройка etcd конфига
```bash
sudo mkdir -p /etc/etcd - создаем папку
sudo touch /etc/etcd/etcd.conf
sudo chown root:root /etc/etcd/etcd.conf
sudo chmod 644 /etc/etcd/etcd.conf
sudo nano /etc/etcd/etcd.conf
```

![etcdconfig](etcd_config.png)

6.11. Cервисный файл, одинаковый для всех нод кластера

![etcdservice](etcd_service.png)

Проделываем на всех 3 нодах, подставляя данные настраиваемой ноды

и запускаем etcd на всех 3 нодах
```bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

Проверяем
```bash
sudo systemctl status etcd
```
![etcdcluster](etcd_cluster.png)

ETCD кластер готов, можно продолжать, но в дополнение установим docker контейнер для удобного отображения etcd и его данных - etcdkeeper. Для наглядности добавим какой-нибудь тестовый ключи через команду etcd put

![etcdKeeper](etcd_keeper.png)

Отлично, etcd настроен + gui через контейнер для удобства отображения ключей.


## 8. Установка PostgreSQL
Не буду расписывать установку субд на всех 3 нодах кластера, так как это тривиальная задача, уточню лишь, что после установки нужно остановить службу PostgreSQL и удалить директорию с данными main, для последующей настройки Patroni.
![postgress](postgress.png)


## 9. Установка Patroni
Создание структуры директорий
```bash
sudo mkdir -p /etc/patroni - файл конфига
sudo mkdir /var/log/patroni/ - файл логов
sudo mkdir -p /var/lib/patroni - метаданные и состояние
sudo mkdir -p /opt/patroni/venv - Создаем корневую директорию Patroni
```
Очистка PostgreSQL (если есть существующий кластер)
```bash
sudo rm -rf /var/lib/postgresql/17/main - Patroni ожидает **пустую** директорию данных, чтобы создать новый кластер
```
Создание пользователя и группы Patroni и назначаем права
```bash
sudo groupadd -r patroni - системную группу patroni
```

Настройка прав
```bash
sudo chown -R patroni:patroni /etc/patroni
sudo chown -R patroni:patroni /opt/patroni
sudo chown -R patroni:patroni /var/lib/patroni
sudo chown -R patroni:patroni /var/log/patroni
```
Права на директории Patroni
```bash
sudo chmod 755 /etc/patroni
sudo chmod 755 /opt/patroni
sudo chmod 755 /var/lib/patroni
sudo chmod 755 /var/log/patroni
```


Создание и настройка директории PostgreSQL
```bash
sudo mkdir -p /var/lib/postgresql/17/main
sudo chown -R patroni:patroni /var/lib/postgresql/17/main
sudo chmod 700 /var/lib/postgresql/17/main
```

Добавляем patroni в группу postgres (для доступа к бинарникам)
```bash
sudo usermod -a -G postgres patroni
```

Создание виртуального окружения
```bash
sudo -u patroni python3 -m venv /opt/patroni/venv
```
Получаем ошибку
![enverror](pythno_env_error.png)

Нет python3-venv, установиливаем пакет.
```bash
sudo apt update
sudo apt install python3-venv python3-pip -y
```
После утсановки пробуем еще раз
```bash
sudo -u patroni python3 -m venv /opt/patroni/venv
```
Ошибка исчезла

Установка Patroni в виртуальное окружение
```bash
sudo -u patroni /opt/patroni/venv/bin/pip install --upgrade pip setuptools wheel
```
![pack1](python_pack_1.png)
```bash
sudo -u patroni /opt/patroni/venv/bin/pip install patroni[etcd3]
```
![pack2](python_pack_2.png)

```bash
sudo -u patroni /opt/patroni/venv/bin/pip install etcd3 psycopg2-binary
```
![pack3](python_pack_3.png)


Итоговый список пакетов для кластера Patroni
* etcd3
* patroni
* psycopg2-binary

Создание конфигурационного файла Patroni
```bash
sudo touch /etc/patroni/patroni.yml
sudo nano /etc/patroni/patroni.yml
```

```yml
# ================================================================
# КОНФИГУРАЦИЯ PATRONI
# ================================================================
# Файл: /etc/patroni/patroni.yml
# Описание: Управление PostgreSQL кластером через etcd

# --- ОСНОВНЫЕ НАСТРОЙКИ КЛАСТЕРА ---

# scope: Уникальное имя кластера (должно быть одинаковым на всех нодах)
scope: postgres-cluster

# namespace: Префикс в etcd для хранения данных Patroni
namespace: /db/

# name: Уникальное имя текущей ноды (разное на каждой ноде!)
name: pg-replica2

# --- НАСТРОЙКИ REST API ---
# Patroni предоставляет REST API для мониторинга и управления

restapi:
  # listen: Адрес и порт для прослушивания API
  listen: 0.0.0.0:8008
  # connect_address: Адрес, который сообщается другим нодам
  connect_address: 192.168.244.133:8008

# --- НАСТРОЙКИ ETCD (DCS) ---
# etcd используется как распределенное хранилище конфигурации

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 100
        max_wal_senders: 10
        wal_level: replica
        hot_standby: on
        password_encryption: scram-sha-256

etcd3:
  # hosts: Список всех нод etcd кластера
  hosts:
    - 192.168.244.131:2379
    - 192.168.244.132:2379
    - 192.168.244.133:2379

# --- НАСТРОЙКИ POSTGRESQL ---

postgresql:
  # listen: Адрес для прослушивания PostgreSQL
  listen: 192.168.244.133:5432
  # connect_address: Адрес для подключения с других нод
  connect_address: 192.168.244.133:5432
  # data_dir: Директория с данными PostgreSQL
  data_dir: /var/lib/postgresql/17/main
  # bin_dir: Директория с бинарниками PostgreSQL
  bin_dir: /usr/lib/postgresql/17/bin
  # pgpass: Путь к файлу с паролями для подключения
  pgpass: /var/lib/patroni/.pgpass
  use_pg_rewind: true
  remove_data_directory_on_rewind_failure: true

  pg_hba:
    - local all all peer
    - host all all 0.0.0.0/0 scram-sha-256
    - host replication replicator 0.0.0.0/0 scram-sha-256

  # --- АУТЕНТИФИКАЦИЯ В POSTGRESQL ---
  # ВНИМАНИЕ: Это РОЛИ PostgreSQL, а НЕ системные пользователи!
  authentication:
    # superuser: Суперпользователь для управления кластером
    superuser:
      username: postgres
      password: postgres
    # replication: Пользователь для репликации
    replication:
      username: replicator
      password: replicator
    # rewind: Пользователь для pg_rewind
    rewind:
      username: rewind_user
      password: rewind_user
```



Выдать права сразу на файл конфига
```yml
sudo chown patroni:patroni /etc/patroni/patroni.yml
sudo chmod 640 /etc/patroni/patroni.yml
```

Создание файла .pgpass
Создаем файл .pgpass для автоматической аутентификации в PostgreSQL

```yaml
localhost:*:*:postgres:postgres
192.168.244.133:*:*:postgres:postgres
```

Устанавливаем правильные права (только владелец может читать/писать)
```bash
sudo chmod 600 /var/lib/patroni/.pgpass
sudo chown patroni:patroni /var/lib/patroni/.pgpass
```


Создание systemd сервиса
```bash
sudo nano /etc/systemd/system/patroni.service
```

```yml
# Создаем systemd unit файл для Patroni
# systemd - система инициализации и управления сервисами
[Unit]
# Описание сервиса
Description=Patroni HA PostgreSQL Cluster
# Ссылка на документацию
Documentation=https://patroni.readthedocs.io/
# Сервис запускается после сети и etcd
After=network.target etcd.service
# Желательно запустить вместе с etcd
Wants=etcd.service
# Запускаем до PostgreSQL (Patroni управляет PostgreSQL)
Before=postgresql.service

[Service]
# Тип сервиса - простой процесс
Type=simple
# Запуск от пользователя patroni (НЕ postgres!)
User=patroni
Group=patroni
# Рабочая директория
WorkingDirectory=/var/lib/patroni

# Переменные окружения
# PATH - сначала venv, потом системные пути
# Environment="PATH=/opt/patroni/venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
# Environment="PATH=/opt/patroni/venv/bin:/usr/lib/postgresql/17/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Environment="PATH=/opt/patroni/venv/bin:/usr/lib/postgresql/17/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Environment="HOME=/var/lib/patroni"
Environment="PGPASSFILE=/var/lib/patroni/.pgpass"
Environment="PATRONI_SCOPE=postgres-cluster"
Environment="PYTHONUNBUFFERED=1"
Environment="TZ=UTC"

# Команда запуска - полный путь к patroni в venv
ExecStart=/opt/patroni/venv/bin/patroni /etc/patroni/patroni.yml
# Перезагрузка конфигурации (сигнал HUP)
ExecReload=/bin/kill -s USR1 $MAINPID

# Управление процессом
KillMode=mixed
KillSignal=SIGTERM
TimeoutStopSec=60

# Стратегия перезапуска
Restart=on-failure
RestartSec=10
StartLimitBurst=3
StartLimitIntervalSec=60

# Ограничения ресурсов
LimitNOFILE=65536
LimitNPROC=65536
TasksMax=infinity
MemoryHigh=2G
MemoryMax=4G

# Логирование
SyslogIdentifier=patroni
StandardOutput=journal
StandardError=journal

[Install]
# Автозапуск при загрузке системы
WantedBy=multi-user.target
```


Проделываем все шаги на остальных нодах.

Запускаем кластер патрони
```bash
sudo systemctl daemon-reload
sudo systemctl enable patroni
sudo systemctl start patroni
```
И проверяем статус
```bash
sudo systemctl status patroni
```

Кластер запустился, но на 2 нодах, одна из реплик выдала ошибку
![ptrn_error](patroni_error.png)

На pg-master машине
![ptrn_error_mstr](patroni_log_error.png)

На pg-replica1
![ptrn_error_node1](patroni_node_failed.png)

Хотя pg-replica2 запустилась
![ptrn_node2_ok](patroni_cluster_list_2node.png)

Значит дело только в pg-replica1, проверяем.
В логах видим ошибку, что нода с названием pg-master уже запущена.

При проверке конфига на pg-replica1, допустил ошибку, забыл переименовать имя ноды, было указано name:pg-master, а должно быть
pg-replica1, исправляем и перезапускаем.

![ptrn_node3_ok](patroni_cluster_3node.png)

Кластер patroni установлен, все 3 ноды видны и работают.


## 10. Установка и настройка haproxy/keepalived.
Устанавливаем все необходимые пакеты
```bash
sudo apt update && sudo apt install -y haproxy keepalived psmisc
```

Создаем конфиг, одинаковый на всех 3 узлах
```bash
touch /etc/haproxy/haproxy.cfg
```
![hprxy_cnfg](haproxy_config.png)

```bash
Создаем сервис файл для keepalive, на каждой ноде свои настройки. Отличия между узлами — только state и priority
```

![hprxy_service](haproxy_service.png)

Разрешить bind на не-локальный VIP + запуск (на всех 3 узлах)
```bash
echo 'net.ipv4.ip_nonlocal_bind = 1' | sudo tee /etc/sysctl.d/99-haproxy.conf
```

Пробуем запустить
```bash
sudo systemctl enable --now haproxy keepalived
```

И рестартем

```bash
sudo systemctl restart haproxy keepalived
```

Поверяем
![keep_alivd_start](keepalived_start.png)


Зайдем по адресу http://192.168.244.140:7000/ и проверим статистику + что все работает

![hprxy_replica_down](haproxy_gui_replicadown.png)

Но видим что мастер up, а вот реплики down

Чтобы HAProxy мог успешно проверять состояние реплик, нужно добавить разрешающую строку в файл pg_hba.conf
Добавляем и проверяем еще раз

![pghba](pghbaconf.png)

Не помогло, все равно реплики down, капаем дальше.
Заменим в конфиге haproxy в блоке listen postgres_primary, строку httpchk GET /primary на option tcp-check
Проверяем
![haproxy_all_ok](haproxy_gui_allok.png)
работает
Что сделали: option tcp-check проверяет только открытый порт (5432), а httpchk проверял HTTP-ответ от Patroni (порт 8008).
Как только вы переключили проверку на TCP, HAProxy просто убедился, что порт базы открыт, и сразу поднял реплики в UP.

Haproxy настроен.


## 11.Функциональная проверка

11.1. Проверка репликации

На мастере создадим таблицу и запишем данные
![replica_test_insert](replication_test_insert.png)

Читаем на реплике

![replica_test_select](replication_test_select.png)

11.2  Отказ мастера (Automatic Failover)

Останавливаем Patroni и Postgres на текущем мастере (192.168.244.131)
```bash
sudo systemctl stop patroni
```

Видим что лидер сменился

![maser_patron_stop](patroni_cluster_1node_stopped.png)


Вернем ноду в кластер видим что бывший лидер вернулся и стал репликой, бывшая реплика стала мастером

![maser_patron_change](patroni_list_all_node.png)

11.3.Проверка работы HAProxy (балансировка)
Убедиться, что HAProxy распределяет нагрузку, и что он видит все ноды.

Для начала простая проверка через команду

![haproxy_test](haproxy_test.png)


11.4.Тест на Живучесть (Failover для HAProxy)

На мастере выполним несколько итераций запроса к HAProxy через команду

![haproxy_send_signal](haproxy_test_send_signal.png)

После чего остановим патрони на одной из реплик и запусти еще раз проверку HAProxy

![haproxy_send_signal_2_node](haproxy_send_signal_on_2node_test.png)


Ошибок нет, HAProxy шлет запросы только на рабочие ноды

## 12. Docker контейнеры для удобства отображения etcd и patroni кластеров
Будем использовать всего 2 контейнера - etcdkeeper и ivory.

![sacks](stacks.png)

ETCD-keeper

```yml
services:
  etcd-browser:
    image: evildecay/etcdkeeper
    container_name: etcd-browser
    ports:
      - "8080:8080"
    environment:
      - ETCD_ENDPOINTS=http://192.168.244.131:2379,http://192.168.244.132:2379,http://192.168.244.133:2379
    restart: unless-stopped
```

Ivory

```yml
volumes:
  data:
    driver: local

networks:
  network:
    driver: overlay
    attachable: true

services:
  webserver:
    image: 'veegres/ivory:v1.4.1'
    hostname: 'ivory-webserver'
    container_name: ivory_webserver
    networks:
      - network
    ports:
      - '8085:80'
    environment:
      - IVORY_URL_PATH=/ivory
      - TZ=Europe/Moscow
    volumes:
      - data:/opt/ivory/data
    deploy:
      placement:
        constraints:
          - 'node.labels.TAG==ivory'
    restart: always
```

Наш кластер patroni в удобной GUI

![ivory](ivory.png)

И etcd

![etcd_last](etcd_gui_last.png)


## 13. Подытожим

Успешно развёрнут отказоустойчивый кластер PostgreSQL 17

Использовали Patroni, etcd, HAProxy и Keepalived.

Что это нам дает:

Автоматическое переключение мастера.

Репликацию данных между мастером и репликами.

Балансировку нагрузки.

Отказоустойчивость балансировщиков (Keepalived).


## 14. Что не успел/планы на развитие кластера.
* Прикрутить мониторинг и алерты - grafana + zabbix.
* Резервное копирование 
* Автоматизация через ansible
* Нагрузить кластер - pgbench
