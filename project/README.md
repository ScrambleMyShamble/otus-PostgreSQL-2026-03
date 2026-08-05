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
| haproxy | Ubuntu 24.04.3 LTS | Операционная система + ВМ |
| VIP PostgreSQL | 2.2.8 | Отказоустойчивый VIP для HAProxy |

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
![postgresstatus](postgres-status.png)


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
**etcd3              0.12.0**
**patroni            4.1.3**
**psycopg2-binary    2.9.12



Создание конфигурационного файла Patroni
```bash
sudo touch /etc/patroni/patroni.yml
sudo nano /etc/patroni/patroni.yml
```
Вставить код файла

Выдать права сразу на файл конфига
```bash
sudo chown patroni:patroni /etc/patroni/patroni.yml
sudo chmod 640 /etc/patroni/patroni.yml


Создание файла .pgpass
Создаем файл .pgpass для автоматической аутентификации в PostgreSQL

вставить код из файла


Устанавливаем правильные права (только владелец может читать/писать)
```bash
sudo chmod 600 /var/lib/patroni/.pgpass
sudo chown patroni:patroni /var/lib/patroni/.pgpass



Создание systemd сервиса
```bash
sudo nano /etc/systemd/system/patroni.service
вставить код из файла


Проделываем все шаги на остальных нодах

Пробуем запустить кластер патрони, неудачно, смотри ошибку
На pg-master машине
скрин
На pg-replica1
скрин

Хотя pg-replica2 запустилась
скрин

Значит дело только в реплике1, проверяем

Cмотрим патрони_лист, не видим одну из реплик
Смотрим логи на pg-replica1 и видим следующее.
скрин
Проверяем конфиг, видим что в конфиге на pg-replica1, указано name:pg-master, а должно быть
pg-replica1, исправляем и перезапускаем на ноде
успех
скрин с реплики
скрин с мастера с 3 нодами в листе


Кластер patroni установлен, все 3 ноды видны и работают



