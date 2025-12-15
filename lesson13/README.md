Урок "Резервное копирование и восстановление"

Создаю новую БД, схему и 2 таблицы:
```pgsql
CREATE DATABASE otus_bkp;

CREATE SCHEMA backup;

CREATE TABLE backup.test (
	id integer,
	name varchar(30)
);


CREATE TABLE backup.restore (   
        id integer,
        name varchar(30)
); 

INSERT INTO backup.test (id, name)
SELECT generate_series(1, 100),'user_' || generate_series(1, 100);
```

Проверяю данные:
```pgsql
SELECT * FROM backup.test;
```

В ответ получаю:
```pgsql
id  |   name   
-----+----------
   1 | user_1
   2 | user_2
   3 | user_3
   4 | user_4
   5 | user_5
   6 | user_6
   7 | user_7
   8 | user_8
   9 | user_9
................
```

Создаю каталог для бекапа и даю полные права на каталог:
```bash
sudo mkdir -p /opt/shared_data

sudo chmod -R 777 /opt/shared_data
```

Создаю бекап в ранее созданный каталог:
```pgsql
\copy backup.test to '/opt/shared_data/std.sql';
```

Восстанавливаю бекап из таблицы 1 в таблицу 2:
```pgsql
COPY backup.restore (id, name) 
FROM '/opt/shared_data/std.sql' 
DELIMITER E'\t';
```

Проверяю данные в таблице 2:
```pgsql
SELECT * FROM backup.restore;
```

В ответ получаю:
```bash
id  |   name
-----+----------
   1 | user_1
   2 | user_2   
   3 | user_3
   4 | user_4
   5 | user_5
   6 | user_6
   7 | user_7   
   8 | user_8
   9 | user_9
................
```

Создаю бекап с помощью pg_dump:
```bash
pg_dump -U postgres -d otus_bkp \
  -t backup.test -t backup.restore \
  -Fc -f otus_bkp.dump
```

Создаю вторую БД и схему в ней:
```pgsql
CREATE DATABASE otus_restore;

CREATE SCHEMA backup;
```

Восстанавливаю таблицу backup.restore в новой БД:
```bash
pg_restore -U postgres \
  -d otus_restore \
  --table="restore" \
  --schema="backup" \
  /opt/shared_data/otus_bkp.dump
```

Проверяю данные:
```pgsql
SELECT * FROM backup.restore;
```

В ответ получаю:
```bash
id  |   name   
-----+----------
   1 | user_1
   2 | user_2
   3 | user_3
   4 | user_4
   5 | user_5
   6 | user_6
   7 | user_7
   8 | user_8
   9 | user_9
.................
```
