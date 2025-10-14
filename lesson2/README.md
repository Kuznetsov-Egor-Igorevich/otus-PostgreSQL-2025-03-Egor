Скачиваю образ postgres:
```bash
sudo docker pull postgres
```


Создаю папку на сервере:
```bash
sudo mkdir /var/lib/postgres
```

Создаю контейнер с монтированной папкой:
```bash
sudo docker run --name otus_lesson_2 \
  -e POSTGRES_USER=otus \
  -e POSTGRES_PASSWORD=otus \
  -e POSTGRES_DB=otus \
  -p 5432:5432 \
  -v /var/lib/postgres:/var/lib/postgresql/18/docker \
  -d \
  postgres
```

Создаю второй контейнер для клиента:
```bash
sudo docker run --name otus_lesson_2_1 \
  -e POSTGRES_USER=otus \
  -e POSTGRES_PASSWORD=otus \
  -e POSTGRES_DB=otus \
  -d \
  postgres
```


подключение к БД клиента:
```bash
sudo docker exec -it otus_lesson_2_1 psql -h localhost -U otus -p 5432
```

Из БД клиента подключаюсь к БД сервера:
```pgsql
\c otus otus 100.113.48.82 5432
```

Создаю тестовую таблицу и наполняю ее данными:
```pgsql
CREATE TABLE lesson2(
  id SERIAL,
  name TEXT);

INSERT INTO lesson2(name)
VALUES('Egor');
```



Подключаюсь с ноутбука к серверу postgres:
```bash
psql -h 100.113.48.82 -p 5432 -U otus -d otus
```

Проверяю наличие таблицы lesson2:
```pgsql
SELECT * FROM lesson2;
```

Таблица на месте.

Останавливаю и удаляю контейнер с сервером:
```bash
sudo docker stop 59abe42cbcd4 && sudo docker rm 59abe42cbcd4 
```

Создаю новый контейнер:
```bash
sudo docker run --name otus_lesson_2 \
  -e POSTGRES_USER=otus \
  -e POSTGRES_PASSWORD=otus \
  -e POSTGRES_DB=otus \  
  -p 5432:5432 \
  -v /var/lib/postgres:/var/lib/postgresql/18/docker \
  -d \
  postgres
```

Проверяю наличие таблицы lesson2:
```pgssql
SELECT * FROM lesson2;
```
Таблица на месте, теперь проверю файлы в папке /var/lib/postgres:
```pgsql
sudo ls -a /var/lib/postgres
```

В папке следующие файлы и каталоги:
.     global	    pg_hba.conf    pg_multixact  pg_serial     pg_stat_tmp  pg_twophase  pg_xact	       postmaster.opts
..    pg_commit_ts  pg_ident.conf  pg_notify	 pg_snapshots  pg_subtrans  PG_VERSION	 postgresql.auto.conf
base  pg_dynshmem   pg_logical	   pg_replslot	 pg_stat       pg_tblspc    pg_wal	 postgresql.conf

