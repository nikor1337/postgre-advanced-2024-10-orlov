# Установка, настройка и тестирование Citus для OLAP
## Установка Citus на 3-х нодовый кластер описана [здесь](https://github.com/nikor1337/postgre-advanced-2024-10-orlov/blob/main/HW16-citus/HW16.md)
## Настройка citus для OLAP:
### Создаем распределенную таблицу:
    -- Step 1: Create table with composite primary key
    CREATE TABLE events (
        event_id bigserial,
        event_time timestamp,
        user_id bigint,
        data jsonb,
        PRIMARY KEY (user_id, event_id)
    );
    
    -- Step 2: Distribute the table
    SELECT create_distributed_table('events', 'user_id');
    
    -- Step 3: Insert test data
    INSERT INTO events (event_time, user_id, data)
    VALUES (NOW(), 1, '{"action": "login"}'),
           (NOW(), 2, '{"action": "purchase"}');
    
    -- Step 4: Verify distribution
    SELECT * FROM events;
### Проверяем, что таблица появилась на worker нодах:
    nick@orlov-citus-worker1:~$ sudo su - postgres -c psql
    psql (16.7 (Ubuntu 16.7-1.pgdg22.04+1))
    Type "help" for help.
    
    postgres=# 
    postgres=# 
    postgres=# \dt
             List of relations
     Schema |  Name  | Type  |  Owner   
    --------+--------+-------+----------
     public | events | table | postgres
    (1 row)
### Загружаем данные в таблицу:
    postgres=# INSERT INTO events (event_time, user_id, data)
    SELECT NOW(), generate_series(1, 20000000), '{"action": "login"}';
    INSERT 0 990001
### Тестируем OLAP запросы:
#### С выключенным поколоночным хранением:
    postgres=# SELECT user_id, COUNT(*) FROM events GROUP BY user_id;
    Time: 961.604 ms
#### Включаем поколоночное хранение и повторно выполняем этот же запрос:
    SELECT alter_table_set_access_method('events', 'columnar');
    postgres=# SELECT user_id, COUNT(*) FROM events GROUP BY user_id;
    Time: 929.197 ms