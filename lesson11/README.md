# Урок "Секционирование"

Создаю копию таблицы bookings с партицированием по диапазону поля 'book_date':
```pgsql
CREATE TABLE bookings_parted (
	book_ref char(6),
	book_date timestamptz,
	total_amount numeric(10, 2),
	
	PRIMARY KEY(book_ref, book_date)
)
PARTITION BY RANGE(book_date);
```

Создаю партиции подтаблиц таблицы bookings_parted по месяцам (так как начальный диапазон с 2026-03-01 02:59:58.993 +0300 по 2025-09-01 03:00:06.265 +0300):
```pgsql
CREATE TABLE bookings.bookings_parted_2025_09 PARTITION OF bookings.bookings_parted FOR VALUES FROM ('2025-09-01 00:00:00+03') TO ('2025-10-01 00:00:00+03');
CREATE TABLE bookings.bookings_parted_2025_10 PARTITION OF bookings.bookings_parted FOR VALUES FROM ('2025-10-01 00:00:00+03') TO ('2025-11-01 00:00:00+03');
CREATE TABLE bookings.bookings_parted_2025_11 PARTITION OF bookings.bookings_parted FOR VALUES FROM ('2025-11-01 00:00:00+03') TO ('2025-12-01 00:00:00+03');
CREATE TABLE bookings.bookings_parted_2025_12 PARTITION OF bookings.bookings_parted FOR VALUES FROM ('2025-12-01 00:00:00+03') TO ('2026-01-01 00:00:00+03');
CREATE TABLE bookings.bookings_parted_2026_01 PARTITION OF bookings.bookings_parted FOR VALUES FROM ('2026-01-01 00:00:00+03') TO ('2026-02-01 00:00:00+03');
CREATE TABLE bookings.bookings_parted_2026_02 PARTITION OF bookings.bookings_parted FOR VALUES FROM ('2026-02-01 00:00:00+03') TO ('2026-03-01 00:00:00+03');
CREATE TABLE bookings.bookings_parted_2026_03 PARTITION OF bookings.bookings_parted FOR VALUES FROM ('2026-03-01 00:00:00+03') TO ('2026-04-01 00:00:00+03');
```

Так же создаю партцию для граничных случаев:
'''pgsql
CREATE TABLE bookings.bookings_parted_default 
PARTITION OF bookings.bookings_parted DEFAULT;
```

Мигрирую данные из основной таблицы bookings, в головную партицированную:
```pgsql
INSERT INTO bookings_parted(book_ref, book_date, total_amount)
SELECT * FROM bookings;
```


Проверю разнесенные данные и отдельно таблицу bookings_parted_default.
Во всех партициях отображаются актуальные данные, таблицы bookings_parted_default пуста, что говорит о том что все мигрировалось верно.

Анализурую запрос к таблице bookings которая не партицирована:
```pgsql
EXPLAIN ANALYZE 
SELECT * FROM bookings
WHERE book_date::date BETWEEN '2025-11-01' AND '2025-11-30';
```

В ответ получаю:
```text
Gather  (cost=1000.00..38821.98 rows=12434 width=21) (actual time=1.318..187.471 rows=410680.00 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  Buffers: shared read=15856 written=28
  ->  Parallel Seq Scan on bookings  (cost=0.00..36578.58 rows=5181 width=21) (actual time=0.255..162.470 rows=136893.33 loops=3)
        Filter: (((book_date)::date >= '2025-11-01'::date) AND ((book_date)::date <= '2025-11-30'::date))
        Rows Removed by Filter: 692010
        Buffers: shared read=15856 written=28
Planning Time: 0.108 ms
Execution Time: 202.409 ms
```

Теперь делаю этот же запрос но к партицированной таблице:
```pgsql
EXPLAIN ANALYZE
SELECT * FROM bookings_parted_2025_11
WHERE book_date::date BETWEEN '2025-11-01' AND '2025-11-30';
```

В ответ получаю:
```text
Gather  (cost=1000.00..8652.83 rows=2053 width=21) (actual time=0.444..78.936 rows=410680.00 loops=1)
  Workers Planned: 1
  Workers Launched: 1
  Buffers: shared hit=2616
  ->  Parallel Seq Scan on bookings_parted_2025_11  (cost=0.00..7447.53 rows=1208 width=21) (actual time=0.022..50.827 rows=205340.00 loops=2)
        Filter: (((book_date)::date >= '2025-11-01'::date) AND ((book_date)::date <= '2025-11-30'::date))
        Buffers: shared hit=2616
Planning Time: 0.111 ms
Execution Time: 93.324 ms
```

Скорость сушественно возрасла.
