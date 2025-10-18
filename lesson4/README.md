#Урок "Логический уровень PostgreSQL"

Подключаюсь к серверу:
```bash
ssh egor@100.113.48.82
```

Подключаюсь к БД через psql:
```bash
sudo -u postgres psql
```
Создаю новую базу:
```pgsql
CREATE DATABASE testdb;
```
Подключаюсь к новой базе:
```bash
\c testdb;
```
Создаю новую схему, таблицу и наполняю таблицу:
```pgsql
CREATE SCHEMA testnm;
CREATE TABLE testnm.t1(
    c1 INTEGER
);
INSERT INTO testnm.t1(c1)
VALUES(1);
```

Создаю новую роль и даю разрешения:
```pgsql
CREATE ROLE readonly;
GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
```

Создаю нового пользователя и даю ему роль:
```pgsql
CREATE USER testread PASSWORD 'test123';
GRANT readonly TO testread;
'''

Подключаюсь к БД от лица нового пользователя и пытаюсь сделать выборку данных:
```pgsql
psql -h localhost -U testread -d testdb
SELECT * FROM testnm.t1;
```

В ответе вернулась одна строка.
Это связано с тем что изначально я создавал таблицу с указанием схемы, так как схема по умолчанию **public**
Для того чтобы изменить схему по умолчанию пригодится:
```pgsql
SELECT set_config('search_path', 'tesnnm', FALSE);
```

Создаю новую таблицу под пользователем testread:
```pgsql
CREATE TABLE testnm.t2(
    c1 INTEGER
);

Отображается ошибка "нет доступа к схеме testnm".
При попытке сменить search_path **SELECT set_config('search_path', 'tesnnm', FALSE);**, отображается ошибка "нет доступа к схеме testnm".

Выдам роле readonly права на создание в схеме **testnm**, из первой сессии под postgres:
```pgsql
GRANT CREATE ON SCHEMA testnm TO readonly;
```

Повторю создание таблицы:
```pgsql
CREATE TABLE testnm.t2( 
    c1 INTEGER
);
```

Таблица успешно создалась, тоже самое произошло и с таблицей **t3**
