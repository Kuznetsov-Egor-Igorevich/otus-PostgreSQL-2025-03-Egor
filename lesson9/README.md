# Урок "Выборка данных, виды join'ов. Применение и оптимизация."

Для имитации данных я установил тестовую БД **demo** скачанную тут: https://edu.postgrespro.ru/demo-20250901-1y.sql.gz

Модель данных расположена тут: https://edu.postgrespro.ru/demo-20250901-1y.sql.gz

Осуществляю простой JOIN:
```pgsql
SELECT 
	b.book_ref,
	b.total_amount,
	t.ticket_no,
	t.passenger_name
FROM bookings AS b
JOIN tickets AS t USING(book_ref)
WHERE b.book_date >= '2025-03-01'
```

В результате отобразились данные которые есть и в первой и во второй таблице

Осуществляю левосторонний JOIN:
```pgsql
SELECT 
	b.book_ref,
	b.total_amount,
	t.ticket_no,
	t.passenger_name
FROM bookings AS b
LEFT JOIN tickets AS t USING(book_ref)
WHERE b.book_date >= '2025-03-01'
```

В результате остались все данные с первой таблицы, если во второй таблице нет данных, то подтягивается NULL

Осуществляю правосторонний JOIN:
```pgsql
SELECT
        b.book_ref,
        b.total_amount,
        t.ticket_no,
        t.passenger_name
FROM bookings AS b
RIGHT JOIN tickets AS t USING(book_ref)
WHERE b.book_date >= '2025-03-01'
```

В результате остались все данные со второй таблицы, если в первой таблице нет данных, то подтягивается NULL


Осуществляю полный JOIN:
```pgsql
SELECT
        b.book_ref,
        b.total_amount,
        t.ticket_no,
        t.passenger_name
FROM bookings AS b
FULL JOIN tickets AS t USING(book_ref)
WHERE b.book_date >= '2025-03-01'
```

В результате остались все данные c первой со второй таблицы, если в какой-то таблице нет данных, то подтягивается NULL

Осуществляю перекрестный JOIN:
```pgsql
SELECT 
	b.book_ref,
	b.total_amount,
	t.ticket_no,
	t.passenger_name
FROM bookings AS b
CROSS JOIN tickets AS t
WHERE b.book_date >= '2025-03-01'
```

В результате на каджую запись из первой таблицы добавились все  записи из второй таблицы.

 
