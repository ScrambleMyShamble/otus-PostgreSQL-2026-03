# Работа vacuum и autovacuum в Postgresql 17.
## Задание и цели
* Понять как работает vacuum и autovacuum, их отличие
* Что такое многоверсионность и как она влияет на использование места на диске

## 1. Окружение
* Vmvare с ОС ubuntu
* Docker rонтейнер с Postgresql 17
* Утилита pgbench для прогона тестов
* Подклчюченный к виртуальной машине Dbeaver

## 2. Подготовка БД через pgbench

```sql
root@2457467c1dd5:/# su - postgres -c "pgbench -i postgres"
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
vacuuming...                                                                              
creating primary keys...
done in 0.46 s (drop tables 0.01 s, create tables 0.02 s, client-side generate 0.23 s, vacuum 0.10 s, primary keys 0.09 s).
```
