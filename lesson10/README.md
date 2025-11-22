# Урок "Виды индексов. Работа с индексами и оптимизация запросов"

Создаю и наполняю таблицу 'index':
```pgsql
CREATE TABLE INDEX (
	id INTEGER,
	first_name VARCHAR(50),
	last_name VARCHAR(50)
);

INSERT INTO otus.index (id, first_name, last_name)
WITH RECURSIVE numbers AS (
    SELECT 1 as id
    UNION ALL
    SELECT id + 1
    FROM numbers
    WHERE id < 50000
)
SELECT 
    id,
    CASE (id % 10)
        WHEN 0 THEN 'Ivan'
        WHEN 1 THEN 'Maria'
        WHEN 2 THEN 'Alexey'
        WHEN 3 THEN 'Olga'
        WHEN 4 THEN 'Dmitry'
        WHEN 5 THEN 'Anna'
        WHEN 6 THEN 'Sergey'
        WHEN 7 THEN 'Elena'
        WHEN 8 THEN 'Pavel'
        WHEN 9 THEN 'Natalia'
    END as first_name,
    CASE (id % 8)
        WHEN 0 THEN 'Ivanov'
        WHEN 1 THEN 'Petrov'
        WHEN 2 THEN 'Sidorov'
        WHEN 3 THEN 'Kuznetsov'
        WHEN 4 THEN 'Smirnov'
        WHEN 5 THEN 'Popov'
        WHEN 6 THEN 'Volkov'
        WHEN 7 THEN 'Fedorov'
    END as last_name
FROM numbers;
```

Смотрю план запрос на  выборку по одному значению:
```pgsql
EXPLAIN ANALYZE
SELECT id FROM otus.INDEX
WHERE id = 10;
```

В ответ получаю:
```text
Seq Scan on index  (cost=0.00..424.88 rows=46 width=4) (actual time=0.044..7.164 rows=1.00 loops=1)
  Filter: (id = 10)
  Rows Removed by Filter: 49999
  Buffers: shared hit=309
Planning Time: 0.078 ms
Execution Time: 7.187 ms
```

Создаю индекс на поле id:
```pgsql
CREATE INDEX otus_test ON otus.INDEX (id);
```


Смотрю план запрос на  выборку по одному значению:
```pgsql
EXPLAIN ANALYZE
SELECT id FROM otus.INDEX
WHERE id = 10;
```

В ответ получаю:
```text
Index Only Scan using otus_test on index  (cost=0.29..4.31 rows=1 width=4) (actual time=0.099..0.101 rows=1.00 loops=1)
  Index Cond: (id = 10)
  Heap Fetches: 0
  Index Searches: 1
  Buffers: shared hit=1 read=2
Planning:
  Buffers: shared hit=18 read=1
Planning Time: 0.418 ms
Execution Time: 0.127 ms
```

Скорость существенно возрасла.

Удаляю индекс:
```pgsql
DROP INDEX IF EXISTS otus_test;
```


Смотрю план запрос на  выборку по имени:
```pgsql
EXPLAIN ANALYZE
SELECT id FROM otus.INDEX
WHERE first_name = 'Anna';
```

В ответ получаю:
```
Seq Scan on index  (cost=0.00..934.00 rows=4887 width=4) (actual time=0.039..6.732 rows=5000.00 loops=1)
  Filter: ((first_name)::text = 'Anna'::text)
  Rows Removed by Filter: 45000
  Buffers: shared hit=309
Planning:
  Buffers: shared hit=17 dirtied=1
Planning Time: 0.321 ms
Execution Time: 7.086 ms
```


Создаю индекс для полнотекстового поиска:
```pgsql
CREATE INDEX users_first_name_fts_idx ON otus.INDEX
USING gin(to_tsvector('english', first_name));
```

Смотрю план запрос на  выборку по имени:
```pgsql
EXPLAIN ANALYZE
SELECT * FROM otus.index 
WHERE to_tsvector('english', first_name) @@ to_tsquery('english', 'Anna');
```

В ответ получаю:
```
Bitmap Heap Scan on index  (cost=9.83..382.16 rows=250 width=17) (actual time=1.416..3.283 rows=5000.00 loops=1)
  Recheck Cond: (to_tsvector('english'::regconfig, (first_name)::text) @@ '''anna'''::tsquery)
  Heap Blocks: exact=309
  Buffers: shared hit=312
  ->  Bitmap Index Scan on users_first_name_fts_idx  (cost=0.00..9.77 rows=250 width=0) (actual time=1.298..1.298 rows=5000.00 loops=1)
        Index Cond: (to_tsvector('english'::regconfig, (first_name)::text) @@ '''anna'''::tsquery)
        Index Searches: 1
        Buffers: shared hit=3
Planning:
  Buffers: shared hit=24 read=1
Planning Time: 0.483 ms
Execution Time: 3.819 ms
```

Скорость так же возросла.


Удаляю индекс:
```pgsql
DROP INDEX users_first_name_fts_idx;
```

Делаю выборку по фамилии:
```pgsql
EXPLAIN ANALYZE
SELECT id FROM otus.INDEX
WHERE last_name = 'Volkov';
```

В ответ получаю:
```
Seq Scan on index  (cost=0.00..934.00 rows=6255 width=4) (actual time=0.024..7.099 rows=6250.00 loops=1)
  Filter: ((last_name)::text = 'Volkov'::text)
  Rows Removed by Filter: 43750
  Buffers: shared hit=309
Planning Time: 0.082 ms
Execution Time: 7.548 ms
```

Создаю индекс с функцией на фамилию:
```pgsql
CREATE INDEX last_name_lower ON otus.index(lower(last_name));
```

Делаю выборку по фамилии:
```pgsql
EXPLAIN ANALYZE
SELECT id FROM otus.INDEX
WHERE lower(last_name) = 'Volkov';
```

В ответ получаю:
```
Bitmap Heap Scan on index  (cost=6.23..316.68 rows=250 width=4) (actual time=0.156..0.157 rows=0.00 loops=1)
  Recheck Cond: (lower((last_name)::text) = 'Volkov'::text)
  Buffers: shared read=2
  ->  Bitmap Index Scan on last_name_lower  (cost=0.00..6.17 rows=250 width=0) (actual time=0.144..0.144 rows=0.00 loops=1)
        Index Cond: (lower((last_name)::text) = 'Volkov'::text)
        Index Searches: 1
        Buffers: shared read=2
Planning:
  Buffers: shared hit=2
Planning Time: 0.151 ms
Execution Time: 0.185 ms
```

Скорость так же возросла.

Удаляю индекс:
```pgsql
DROP INDEX last_name_lower;
```


Делаю выборку по ид и имени:
```pgsql
EXPLAIN ANALYZE
SELECT * FROM otus.INDEX
WHERE id = 6 AND first_name = 'Sergey';
```

В ответ получаю:
```
Seq Scan on index  (cost=0.00..1059.00 rows=1 width=17) (actual time=0.021..5.583 rows=1.00 loops=1)
  Filter: ((id = 6) AND ((first_name)::text = 'Sergey'::text))
  Rows Removed by Filter: 49999
  Buffers: shared hit=309
Planning Time: 0.101 ms
Execution Time: 5.603 ms
```

Создаю составной индекс:
```pgsql
CREATE INDEX id_first_name ON otus.index(id, first_name);
```


Делаю выборку по ид и имени:
```pgsql
EXPLAIN ANALYZE
SELECT * FROM otus.INDEX
WHERE id = 6 AND first_name = 'Sergey';
```

В ответ получаю:
```Index Scan using id_first_name on index  (cost=0.29..8.31 rows=1 width=17) (actual time=0.098..0.100 rows=1.00 loops=1)
  Index Cond: ((id = 6) AND ((first_name)::text = 'Sergey'::text))
  Index Searches: 1
  Buffers: shared hit=1 read=2
Planning:
  Buffers: shared hit=18 read=1
Planning Time: 0.388 ms
Execution Time: 0.128 ms
```

В общем скорость всегда выше.
