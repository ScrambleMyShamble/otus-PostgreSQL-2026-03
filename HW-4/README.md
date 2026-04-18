# PostgreSQL 17: Нагрузочное тестирование
## Задание и цели
* Развернуть виртуальную машину
* Поставить на неё PostgreSQL 17 и прогнать нагрузочное тестирование с дефолтными настройками
* Настроить кластер PostgreSQL 17 на максимальную производительность
* Нагрузить кластер через утилиту pgbench
* Подвести итоги, что подкрутили, что получили и почему, сравнить до и после

# 1. Разворачиваем PostgreSQL 17
Поднимаем ранее используемый контейнер с PostgreSQL 17 через стэк

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

## 2. Создаем БД pgbench_test через контейнер. Прогон без подкручивания и изменения настроек, 'как есть'. Запоминаем статистику.
```sql
root@7ca7365ed9d8:/# psql -U postgres
psql (17.9 (Debian 17.9-1.pgdg13+1))
Type "help" for help.

postgres=# CREATE DATABASE pgbench_test;
CREATE DATABASE
```

## 3. Инициализируем pgbench
```sql
root@7ca7365ed9d8:/# pgbench -U postgres -i -s 10 pgbench_test
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
vacuuming...                                                                                
creating primary keys...
done in 8.14 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 5.91 s, vacuum 0.49 s, primary keys 1.71 s).
```

<img width="568" height="255" alt="image" src="https://github.com/user-attachments/assets/9120596f-9e36-4c8c-9619-14c0a64f99a6" />

## 4. Прогон с фиксированными настройками.
```sql
pgbench -U postgres -c 10 -j 4 -T 60 pgbench_test
```
Где 
&nbsp;

-c = Количество клиентов
&nbsp;

-j = Количество потоков
&nbsp;

-T = Время в секундах

## Результат
```sql
root@7ca7365ed9d8:/# pgbench -U postgres -c 10 -j 4 -T 60 pgbench_test
pgbench (17.9 (Debian 17.9-1.pgdg13+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 10
number of threads: 4
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 45186
number of failed transactions: 0 (0.000%)
latency average = 13.289 ms
initial connection time = 39.164 ms
tps = 752.517764 (without initial connection time)
```

Особенно нас интересует строчка tps = 752.517764 (without initial connection time). 

# 5. Настройка для максимальной производительности
## Характеристики тестовой системы:
* Диск: sda 100G disk VMware Virtual S. Тип 1, т.е. HDD Диск
* CPU: 2 ядра
* RAM:
&nbsp;

| total       | used        |    free    |  shared     | buff/cache   | available    |
|-------------|-------------|-------------|-------------|-------------|-------------|
|    7.5Gi    |   1.5Gi     | 4.1Gi      |  54Mi       | 2.2Gi        | 6.0Gi        |

### Интересующие нас настройки postgresql, которые сейчас выставлены по дефолту:
```sql
SHOW shared_buffers;
128MB
SHOW effective_cache_size;
4GB
SHOW work_mem;
4MB
SHOW wal_buffers;
4MB
SHOW max_connections;
100
```

### Давайте, отталкиваясь от системы, разгоним нашего бедолагу, выставив нужные значения в конфигурационном файле Postgresql.

* Кэширует данные в RAM
<img width="583" height="71" alt="image" src="https://github.com/user-attachments/assets/e7ef760d-58c1-4c43-aa91-e9f3acd4c521" />
&nbsp;

* Планы запросов
<img width="521" height="37" alt="image" src="https://github.com/user-attachments/assets/d5d1860d-3e8a-4447-ad0b-d2b2bb5d4d9f" />
&nbsp;

* Увеличивает память для сортировок/хешей
<img width="936" height="79" alt="image" src="https://github.com/user-attachments/assets/9de2435a-a8ea-4355-abad-c196e749a0ac" />
&nbsp;

* Буфер для WAL перед записью
<img width="1021" height="49" alt="image" src="https://github.com/user-attachments/assets/88781d0f-19c1-4c4f-a465-b6b55a5d2225" />
&nbsp;

* Не ждёт записи WAL на диск
<img width="774" height="54" alt="image" src="https://github.com/user-attachments/assets/1ba61a59-71fd-4581-8655-c0a54eb61ace" />
&nbsp;

* Растягивает запись "грязных" страниц на диск
<img width="961" height="48" alt="image" src="https://github.com/user-attachments/assets/97675e43-7273-456b-87c4-156e400ad49d" />
&nbsp;

* Отключает репликацию
<img width="781" height="49" alt="image" src="https://github.com/user-attachments/assets/33f5e6db-9597-4e86-82ca-63cf31d5044d" />
&nbsp;

* Сброс данных на диск
<img width="892" height="71" alt="image" src="https://github.com/user-attachments/assets/a4d7c81f-8cdb-42ea-a81f-2147c0d27731" />
&nbsp;

* Не дублирует страницы в WAL
<img width="845" height="53" alt="image" src="https://github.com/user-attachments/assets/2580f816-9c1e-4353-b949-947053554af9" />
&nbsp;

Применяем настройки перезагрузив контейнер с Postgresql 17, через stop-start stack

<img width="1229" height="744" alt="image" src="https://github.com/user-attachments/assets/8bd3fa62-999e-42e9-aaa3-d0ba48c16c50" />

Проверяем на примере одного параметра что настрйоки применились

<img width="440" height="775" alt="image" src="https://github.com/user-attachments/assets/6999ede9-21d9-473b-a9f8-c4b4af929e19" />

Успех, можно гнать новые тесты.

# 6. Прогоняем тот же тест, но с новыми настройками
```sql
pgbench -U postgres -c 10 -j 4 -T 60 pgbench_test
```

Результат
```sql
root@2457467c1dd5:/# pgbench -U postgres -c 10 -j 4 -T 60 pgbench_test
pgbench (17.9 (Debian 17.9-1.pgdg13+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 10
number of threads: 4
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 78198
number of failed transactions: 0 (0.000%)
latency average = 7.669 ms
initial connection time = 45.623 ms
tps = 1303.981743 (without initial connection time)
```

## Видим улучшение на ~70% по tps с 752.517764 до 1303.981743.

# 7. Что и почему подкрутили
| Параметр | Почему изменили |
|-------------|-------------|
| fsync = off	                        | PostgreSQL не будет принудительно сбрасывать данные на диск при каждом коммите → HDD не тормозит |
| synchronous_commit = off	          | Не ждёт записи WAL на диск → каждая транзакция быстрее |
| full_page_writes = off	            | Не дублирует страницы в WAL после чекпоинтов → меньше записей на диск |
| shared_buffers = 2GB	              | Кэширует данные в RAM (25% от памяти) → меньше чтений с медленного HDD |
| effective_cache_cache = 4GB	        | Подсказка планировщику: ОС может использовать ~4GB под кэш → выбирает правильные планы запросов |
| work_mem = 32MB	                    | Увеличивает память для сортировок/хешей (было 4MB) → меньше временных файлов на диске |
| wal_buffers = 16MB	                | Больше буфер для WAL перед записью → реже сбрасывается на диск |
| checkpoint_completion_target = 0.9	| Растягивает запись "грязных" страниц на диск → снижает пиковую нагрузку на HDD |
| max_wal_senders = 0	                | Отключает репликацию → не нужна для теста, экономит ресурсы |


# 8. Итоги
Что дало максимальный эффект:
* Отключение fsync и synchronous_commit — самый большой прирост TPS, потому что диск (даже виртуальный) — узкое место.
* Увеличение shared_buffers до 2GB — позволяет кэшировать больше данных в RAM, снижая чтение с диска.
* Увеличение work_mem до 32MB — ускоряет сортировки и агрегации (если тест их использует).

Для тестов производительности на ВМ критично отключать синхронную запись на диск, но в боевой среде это опасно.
Оптимальный баланс: включить synchronous_commit = on, но поднять shared_buffers до 25-40% от RAM и настроить контрольные точки.












