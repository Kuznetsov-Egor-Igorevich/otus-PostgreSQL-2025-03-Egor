# Урок "SQL и реляционные СУБД. Введение в PostgreSQL"

## Введение

В рамках цикла обучения я буду использовать следующие инструменты:
 * Raspberry pi zero 2w - 100.114.47.78;
 * Raspberry pi3 - 100.102.213.117;
 * Raspberry pi5 - 100.93.200.33;
 * Ubuntu 22.04.5 LTS в качестве основного сервера - 100.113.48.82;
 * Собственно настроенный ВПН.
 ..* В дальнейшем я буду указывать **название** хоста без указания **ip**.

Ставлю postgresql на Raspberry pi5:
```bash
ssh egor@100.93.200.33
```

Обновляю репозитории и пакеты:
```bash
sudo apt update && apt upgrade -y
```

Устанавлиаю postgresql:
```bash
sudo apt install postgresql -y
```

Подключаюсь к базе с pi5:
```bash
sudo -u postgres psql
```

Подключаюсь к pi zero 2w:
```bash
ssh egor@100.114.47.78
```

Подключась к pi5 с pi zero 2w:
```bash
ssh egor@100.93.200.33
```

Отключаю автокоммит на pi5 и проверяю:
```pgsql
\set AUTOCOMMIT off
\echo :AUTOCOMMIT
```

Подключаюсь к базе с pi zero 2w:
```bash
sudo -u postgres psql 
```


Отключаю автокоммит на pi zero 2w и проверяю:
```pgsql
\set AUTOCOMMIT off
\echo :AUTOCOMMIT
```

Создаю, наполняю и проверяю таблицу в первой сессии:
```pgsql
CREATE TABLE persons (
	id SERIAL, 
	first_name TEXT, 
	second_name TEXT
);
INSERT INTO persons (first_name, second_name) 
	VALUES('Ivan', 'Ivanov'), ('Petr', 'Petrov');
COMMIT;
```

Проверяю уровень изоляции в каждой сессии:
```pgsql
SHOW TRANSACTION ISOLATION LEVEL;
```

В обоих сессиях отображается **read committed**.

В первой сесси добавляю новую запись:
```pgsql
INSERT INTO persons(first_name, second_name) 
VALUES('Sergey', 'Sergeev');
```

Во второй сессии делаю запрос на получения всех записей из таблицы **persons**, в ответ получаю 2 записи "Ivan Ivanov", "Petr Petrov".
Связано это с тем что в первой сессии транзакция не была закомичена.

Но после того как я делал коммит в первой сессии и при повторном запросе всех данных из второй сессии в таблице **persons**, вернулось +1 запись "Sergey Sergeev".

Меняю уровень изоляции во всех сессиях:
```pgsql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

Проверяю уровень изоляции в каждой сессии:
```pgsql
show transaction isolation level;
```

В каждой сессии установлен новый уровень **repeatable read**

В первой сессии добавляю новую запись:
```pgsql
INSERT INTO persons(first_name, second_name) 
VALUES('Sveta', 'Svetova');
```

Во второй сессии делаю запрос на получения всех записей из таблицы **persons**, в ответ получаю 3 записи "Ivan Ivanov", "Petr Petrov", "Sergey Sergeev".


Связано это с тем что в первой сессии транзакция не была закомичена, а сессия 2 видит только согласованный снимок до начала транзакции.


После того как я сделал коммит в первой сессии и при повторном запросе всех данных из второй сессии в таблице **persons**, вернулось +1 запись "Sveta Svetova".

