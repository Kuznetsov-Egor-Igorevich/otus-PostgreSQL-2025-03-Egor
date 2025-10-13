# Урок "SQL и реляционные СУБД. Введение в PostgreSQL"

## Введение

В рамках цикла обучения я буду использовать следующие инструменты:
 * Raspberry pi zero 2w - 100.114.47.78;
 * Raspberry pi3 - 100.102.213.117;
 * Raspberry pi5 - 100.93.200.33;
 * Ubuntu 22.04.5 LTS в качестве основного сервера - 100.113.48.82;
 * Собственно настроенный ВПН.
  * В дальнейшем я буду указывать **название** хоста без указания **ip**.

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

