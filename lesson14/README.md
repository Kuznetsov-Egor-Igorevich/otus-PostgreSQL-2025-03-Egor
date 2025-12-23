# Урок "Виды и устройство репликации в PostgreSQL. Практика применения"

Подключаюсь к первому инстансу:
```bash
sudo -u postgres psql
```
Включаю параметр для логической репликации:
```bash
ALTER system SET wal_level = logical;
```

Перезагружаю кластер:
```bash
sudo pg_ctlcluster 18 main restart
```

Создаю новые таблицы:
```bash
CREATE TABLE test (i int);
CREATE TABLE test2 (i int);
```

Создаю публикацию для таблицы:
```bash
CREATE PUBLICATION test_pub FOR TABLE test;
```

Подключаюсь ко второму инстансу и включаю параметр для репликации:
```bash
ALTER system SET wal_level = logical;
```
   
Перезагружаю кластер:
```bash
sudo pg_ctlcluster 18 main restart
```

Создаю таблицы на втором инстансе:
```bash
CREATE TABLE test2 (i int);
CREATE TABLE test (i int);
```

Создаю публикацию для таблицы:
```bash
CREATE PUBLICATION test_pub FOR TABLE test2;
```

Со второго инстанса подписываюсь на таблицу test первого инстанса:
```bash
CREATE SUBSCRIPTION test_sub
CONNECTION 'host=100.113.48.82 port=5432 user=postgres password=Password dbname=otus' PUBLICATION test_pub WITH (copy_data = true);
```

С первого инстанса подписываюсь на таблицу test2 второго инстанса:
```bash
CREATE SUBSCRIPTION test_sub
CONNECTION 'host=100.93.200.33 port=5432 user=postgres password=Password dbname=otus' PUBLICATION test_pub WITH (copy_data = true);
```

Создаю таблицы на третьем инстансе:
```bash
CREATE TABLE test (i int);
CREATE TABLE test2 (i int);
```

С третьего инстанса подписываюсь на таблицу test первого инстанса:
```bash
CREATE SUBSCRIPTION test_sub2
CONNECTION 'host=100.113.48.82 port=5432 user=postgres password=Password! dbname=otus' PUBLICATION test_pub WITH (copy_data = true);
```

С третьего инстанса подписываюсь на таблицу test2 второго инстанса:
```bash
CREATE SUBSCRIPTION test_sub3
CONNECTION 'host=100.93.200.33 port=5432 user=postgres password=Rocker69! dbname=otus' PUBLICATION test_pub WITH (copy_data = true);
```

На первом инстансе вставляю данные:
```bash
INSERT INTO test (i)
SELECT generate_series(1, 11);
```

На втором инстансе проверяю таблицу test:
```bash
SELECT * FROM test
```

В ответ получаю:
```text
 i  
----
  1
  2
  3
  4
  5
  6
  7
  8
  9
 10
 11
(11 rows)
```

Тоже самое делаю на третьем инстансе:
```bash
SELECT * FROM test
```

В ответ получаю:
```text
 i
----
  1
  2
  3
  4
  5
  6
  7
  8
  9
 10
 11
(11 rows)
```

На втором инстансе делаю вставку:
```bash
INSERT INTO test2 (i)
SELECT generate_series(1, 11);
```

На первом инстансе проверяю данные:
```bash
SELECT * FROM test2;
```

В ответ получаю:
```text
 i  
----
  1
  2
  3
  4
  5
  6
  7
  8
  9
 10
 11
(11 строк)
```


```bash
SELECT * FROM test
```

В ответ получаю:
```text
 i 
----
  1
  2
  3
  4
  5
  6
  7
  8
  9 
 10
 11
(11 rows)
```


На третьем инстансе включаю параметр для физической репликации:
```bash
ALTER SYSTEM SET wal_level = replicа;
```

На третьем инстансе в pg_hba.conf добавляю подключение для реплики:
```bash
host    replication     postgres        100.102.213.117/10      password
```

На четвертом инстансе проверяю кластеры:
```bash
sudo ls -la /var/lib/postgresql/18/
```

В ответ получаю:
```bash
drwxr-xr-x  3 postgres postgres 4096 Nov 16 12:41 .
drwxr-xr-x  3 postgres postgres 4096 Nov 17 17:50 ..
drwx------ 19 postgres postgres 4096 Nov 17 18:36 main
```

Создаю новый кластер на четвертом инстансе:
```bash
sudo pg_createcluster -d /var/lib/postgresql/18/main2 18 main2
```

Настраиваю на четвертом инстансе pg_hba.conf:
```bash
host    replication     postgres        100.102.213.117/10      password
```

На четвертом инстансе очищаю каталог:
```bash
sudo rm -rf /var/lib/postgresql/18/main2
```

Создаю бекап БД третьего инстанса на четвертом инстансе:
```bash
sudo -u postgres pg_basebackup -h 100.102.213.117 -p 5432 -R -D /var/lib/postgresql/18/main2
```

