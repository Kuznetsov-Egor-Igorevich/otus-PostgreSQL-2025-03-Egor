# Урок "MVCC, vacuum и autovacuum."

Создаю базу данных:
```pgsql
CREATE DATABASE otus;
```

Захожу под пользователем postgres и запускаю pg_bench:
```bash
sudo su postgres
pgbench -i otus
```

В ответ получил:
```bash
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) of pgbench_accounts done (elapsed 0.26 s, remaini                                                                                vacuuming...
creating primary keys...
done in 0.47 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 0.33 s, vacuum 0.07 s, primary keys 0.06 s).
```

Тестирую производительность:
```bash
pgbench -c8 -P 6 -T 60 -U postgres otus
```

В ответ получил:
```bash
progress: 6.0 s, 471.0 tps, lat 16.694 ms stddev 12.162, 0 failed
progress: 12.0 s, 481.1 tps, lat 16.612 ms stddev 11.794, 0 failed
progress: 18.0 s, 492.0 tps, lat 16.275 ms stddev 11.185, 0 failed
progress: 24.0 s, 486.2 tps, lat 16.444 ms stddev 11.225, 0 failed
progress: 30.0 s, 521.3 tps, lat 15.341 ms stddev 10.401, 0 failed
progress: 36.0 s, 474.7 tps, lat 16.849 ms stddev 12.603, 0 failed
progress: 42.0 s, 482.3 tps, lat 16.558 ms stddev 11.358, 0 failed
progress: 48.0 s, 472.8 tps, lat 16.937 ms stddev 12.059, 0 failed
progress: 54.0 s, 489.5 tps, lat 16.331 ms stddev 11.966, 0 failed
progress: 60.0 s, 471.8 tps, lat 16.955 ms stddev 11.383, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 29065
number of failed transactions: 0 (0.000%)
latency average = 16.488 ms
latency stddev = 11.626 ms
initial connection time = 89.481 ms
tps = 484.929253 (without initial connection time)
```

Меняю параметры автовакуума:
```pgsql
ALTER SYSTEM SET autovacuum_max_workers = 10;
ALTER SYSTEM SET autovacuum_naptime = 15;
ALTER SYSTEM SET autovacuum_vacuum_threshold = 25;
ALTER SYSTEM SET autovacuum_vacuum_scale_factor = 0.05;
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = 10;
ALTER SYSTEM SET autovacuum_vacuum_cost_limit = 1000;
```

Перезагружаю кластер:
```bash
sudo systemctl restart postgresql@18-main
```

Запускаю тест производительности по новой:
```bash
pgbench -c8 -P 6 -T 60 -U postgres otus
```

В ответ получаю:
```bash
pgbench (18.0 (Ubuntu 18.0-1.pgdg22.04+3))
starting vacuum...end.
progress: 6.0 s, 466.0 tps, lat 16.771 ms stddev 11.664, 0 failed
progress: 12.0 s, 486.7 tps, lat 16.518 ms stddev 12.099, 0 failed
progress: 18.0 s, 518.3 tps, lat 15.434 ms stddev 10.736, 0 failed
progress: 24.0 s, 485.3 tps, lat 16.484 ms stddev 11.828, 0 failed
progress: 30.0 s, 496.7 tps, lat 16.097 ms stddev 11.585, 0 failed
progress: 36.0 s, 475.0 tps, lat 16.830 ms stddev 11.688, 0 failed
progress: 42.0 s, 481.2 tps, lat 16.624 ms stddev 11.850, 0 failed
progress: 48.0 s, 487.7 tps, lat 16.401 ms stddev 11.306, 0 failed
progress: 54.0 s, 512.8 tps, lat 15.585 ms stddev 11.769, 0 failed
progress: 60.0 s, 489.0 tps, lat 16.363 ms stddev 11.776, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 29400
number of failed transactions: 0 (0.000%)
latency average = 16.298 ms
latency stddev = 11.639 ms
initial connection time = 94.167 ms
tps = 490.621839 (without initial connection time)
```

По сути практически ничего не изменилось, если только повысился чуть-чуть tps, предполагаю что это связано с увелечением рабочих и изменения autovacuum_vacuum_cost_delay

Создаю в базе новую схему и таблицу:
```pgsql
CREATE SCHEMA test;
CREATE TABLE test.test1 (id TEXT);
```

Наполняю таблицу данными и проверяю размер:
```pgsql
INSERT INTO test.test1(id) SELECT 'noname' FROM GENERATE_SERIES(1, 1000000);
SELECT pg_size_pretty(pg_total_relation_size('test.test1'));
```

В ответ получаю 35 MB.

5 раз обновляю данные в таблице:
```pgsql
UPDATE test.test1
SET id = 'noname1'
WHERE id = 'noname'

UPDATE test.test1
SET id = 'noname2'
WHERE id = 'noname1'

UPDATE test.test1
SET id = 'noname3'
WHERE id = 'noname2'

UPDATE test.test1
SET id = 'noname4'
WHERE id = 'noname3'

UPDATE test.test1
SET id = 'noname5'
WHERE id = 'noname4'
```

Проверяю размер таблицы:
```pgsql
SELECT pg_size_pretty(pg_total_relation_size('test.test1'));
```

В ответ получаю 69 MB.

Проверяю кол-во мертвых строчек:
```pgsql
SELECT relname, n_live_tup, n_dead_tup
FROM pg_stat_user_tables WHERE relname = 'test1';
```

В ответ получил relname = test1, n_live_tup = 1 000 000, n_dead_tup = 0

Проверю когда последний раз сработал автовакуум:
```pgsql
SELECT * FROM pg_stat_activity WHERE query ~ 'autovacuum';
```


В ответ получил:
16469	otus	109500		10	postgres	DBeaver 25.1.4 - SQLEditor <Script-15.sql>	100.74.43.33		58998	2025-10-29 20:48:23.713 +0300	2025-10-29 21:20:11.999 +0300	2025-10-29 21:20:12.001 +0300	2025-10-29 21:20:12.001 +0300			active		442689		SELECT * FROM pg_stat_activity WHERE query ~ 'autovacuum'	client backend


Опять 5 раз обновляю данные в таблице:
```pgsql
UPDATE test.test1
SET id = 'noname6'
WHERE id = 'noname5'

UPDATE test.test1
SET id = 'noname7'
WHERE id = 'noname6'

UPDATE test.test1
SET id = 'noname8'
WHERE id = 'noname7'

UPDATE test.test1
SET id = 'noname9'
WHERE id = 'noname8'

UPDATE test.test1
SET id = 'noname10'
WHERE id = 'noname9'
```

Проверяю размер таблицы:
```pgsql
SELECT pg_size_pretty(pg_total_relation_size('test.test1'));
```

В ответ получаю 111 MB.

Отключаю автовакуум:
```pgsql
ALTER TABLE test.test1 SET (autovacuum_enabled = OFF);
```

10 раз обновляю строки в таблице:
```pgsql
UPDATE test.test1
SET id = 'name'

UPDATE test.test1
SET id = 'name1'

UPDATE test.test1
SET id = 'name2'

UPDATE test.test1
SET id = 'name3'

UPDATE test.test1
SET id = 'name4'

UPDATE test.test1
SET id = 'name5'

UPDATE test.test1
SET id = 'name6'

UPDATE test.test1
SET id = 'name7'

UPDATE test.test1
SET id = 'name8'

UPDATE test.test1
SET id = 'name9'

UPDATE test.test1
SET id = 'name10'
```

Смотрю размер таблицы:
```pgsql
SELECT pg_size_pretty(pg_total_relation_size('test.test1'));
```

В ответ получаю 422 MB.

Это связано с тем что создалось очень много мертвых строк, а именно 10 996 059.
