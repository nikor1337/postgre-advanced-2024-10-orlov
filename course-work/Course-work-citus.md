# Установка, настройка и тестирование Citus для OLAP
## Установка Citus на 3-х нодовый кластер описана [здесь](https://github.com/nikor1337/postgre-advanced-2024-10-orlov/blob/main/HW16-citus/HW16.md)
## Настройка citus для OLAP:
### Создаем распределенные таблицы с поколоночным и построчным хранением:
    postgres=# CREATE TABLE events_column (
            event_id bigserial,
            event_time timestamp,
            user_id bigint,
            data jsonb 
        );                                 
    CREATE TABLE
    Time: 5.755 ms
    postgres=# CREATE TABLE events_row (
            event_id bigserial,
            event_time timestamp,
            user_id bigint,
            data jsonb
        );
    CREATE TABLE
    Time: 6.730 ms
    postgres=# SELECT create_distributed_table('events_row', 'user_id');
     create_distributed_table 
    --------------------------
     
    (1 row)
    
    Time: 140.986 ms
    postgres=# SELECT create_distributed_table('events_column', 'user_id');
     create_distributed_table 
    --------------------------
     
    (1 row)
    
    Time: 221.524 ms
    postgres=# 
    postgres=# 
    postgres=# SELECT alter_table_set_access_method('events_column', 'columnar');
    NOTICE:  creating a new table for public.events_column
    NOTICE:  moving the data of public.events_column
    NOTICE:  dropping the old public.events_column
    NOTICE:  renaming the new table to public.events_column
     alter_table_set_access_method 
    -------------------------------
     
    (1 row)
    
    Time: 134.374 ms
    postgres=# CREATE TABLE users (
                   id bigserial, name text
               );
    CREATE TABLE
    postgres=# 
    postgres=# 
    postgres=# SELECT create_distributed_table('users', 'id');
     create_distributed_table 
    --------------------------
     
    (1 row)


### Загружаем данные в таблицы:
    INSERT INTO events_row (event_time, user_id, data)
    SELECT NOW(), generate_series(1, 1000000), '{"action": "login"}'; #10х
    INSERT INTO events_row (event_time, user_id, data)
    SELECT NOW(), generate_series(1, 1000000), '{"action": "logout"}'; #10х
    INSERT INTO events_row (event_time, user_id, data)
    SELECT NOW(), generate_series(1, 1000000), '{"action": "purchase"}'; #10x
    INSERT INTO events_column (event_time, user_id, data)
    SELECT NOW(), generate_series(1, 1000000), '{"action": "login"}'; #10х
    INSERT INTO events_column (event_time, user_id, data)
    SELECT NOW(), generate_series(1, 1000000), '{"action": "logout"}'; #10х
    INSERT INTO events_column (event_time, user_id, data)
    SELECT NOW(), generate_series(1, 1000000), '{"action": "purchase"}'; #10x
    INSERT INTO users (name)                                                            
    SELECT 'user' || generate_series(1, 1000000); #2x

### Размер таблиц:
    postgres=# SELECT logicalrelid AS name,
           pg_size_pretty(citus_table_size(logicalrelid)) AS size
      FROM pg_dist_partition;
         name      |  size   
    ---------------+---------
     events_row    | 2351 MB
     events_column | 91 MB
     users         | 101 MB
    (3 rows)

### Тестируем OLAP запросы:
#### Таблица с поколоночным хранением:
    postgres=# explain analyze select name, data from users join events_column on users.id = events_column.user_id;
                                                                                  QUERY PLAN                                                                                   
    -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
     Custom Scan (Citus Adaptive)  (cost=0.00..0.00 rows=100000 width=64) (actual time=48794.099..51382.950 rows=30000000 loops=1)
       Task Count: 32
       Tuple data received from nodes: 893 MB
       Tasks Shown: One of 32
       ->  Task
             Tuple data received from node: 28 MB
             Node: host=orlov-citus-worker1 port=5432 dbname=postgres
             ->  Hash Join  (cost=1803.73..14859.93 rows=936840 width=42) (actual time=109.647..3170.024 rows=936840 loops=1)
                   Hash Cond: (events_column.user_id = users.id)
                   ->  Custom Scan (ColumnarScan) on events_column_102304 events_column  (cost=0.00..174.65 rows=936840 width=40) (actual time=1.169..942.340 rows=936840 loops=1)
                         Columnar Projected Columns: user_id, data
                   ->  Hash  (cost=1022.77..1022.77 rows=62477 width=18) (actual time=108.126..108.128 rows=62477 loops=1)
                         Buckets: 65536  Batches: 1  Memory Usage: 3929kB
                         ->  Seq Scan on users_102336 users  (cost=0.00..1022.77 rows=62477 width=18) (actual time=0.016..22.724 rows=62477 loops=1)
                 Planning Time: 1.909 ms
                 Execution Time: 4620.375 ms
     Planning Time: 0.866 ms
     Execution Time: 52477.578 ms
    (18 rows)
    
    Time: 52482.674 ms (00:52.483)
#### Таблица с построчным хранением:
    postgres=# explain analyze select name, data from users join events_row on users.id = events_row.user_id;
                                                                           QUERY PLAN                                                                        
    ---------------------------------------------------------------------------------------------------------------------------------------------------------
     Custom Scan (Citus Adaptive)  (cost=0.00..0.00 rows=100000 width=64) (actual time=48643.199..51226.920 rows=30000000 loops=1)
       Task Count: 32
       Tuple data received from nodes: 893 MB
       Tasks Shown: One of 32
       ->  Task
             Tuple data received from node: 28 MB
             Node: host=orlov-citus-worker1 port=5432 dbname=postgres
             ->  Hash Join  (cost=1808.49..33444.25 rows=937590 width=35) (actual time=136.070..3467.225 rows=937590 loops=1)
                   Hash Cond: (events_row.user_id = users.id)
                   ->  Seq Scan on events_row_102248 events_row  (cost=0.00..18743.90 rows=937590 width=33) (actual time=0.036..745.966 rows=937590 loops=1)
                   ->  Hash  (cost=1025.44..1025.44 rows=62644 width=18) (actual time=135.680..135.681 rows=62644 loops=1)
                         Buckets: 65536  Batches: 1  Memory Usage: 3938kB
                         ->  Seq Scan on users_102344 users  (cost=0.00..1025.44 rows=62644 width=18) (actual time=0.018..68.385 rows=62644 loops=1)
                 Planning Time: 0.700 ms
                 Execution Time: 4558.073 ms
     Planning Time: 0.709 ms
     Execution Time: 52323.910 ms
    (17 rows)
    
    Time: 52328.504 ms (00:52.329)