# Урок "Физический уровень PostgreSQL"

Скачиваю postgresql-18:
```bash
sudo apt install postgresql-18
```

Проверяю что кластер запущен:
```bash
sudo -u postgres pg_lsclusters
```

Подключаюсь к БД:
```bash
sudo -u postgres psql
```

Создаю тестовую таблицу и наполняю ее данными:
```pgsql
CREATE TABLE test1(
   id SERIAL,
   name TEXT
);

INSERT INTO test1 (name)
VALUES('Test');
```

Останавливаю работу кластера:
```bash
sudo systemctl stop postgresql@18-main
```

Создаю секционирование:
```bash
sudo parted /dev/sda mklabel gpt
```

Создаю новый раздел на флешке:
```bash
sudo parted -a opt /dev/sda mkpart primary ext4 0% 100%
```

Создаю файловую систему в новом разделе:
```bash
sudo mkfs.ext4 -L sandisk128 /dev/sda1
```

Создаю директорию:
```bash
sudo mkdir -p /mnt/data
```

Вношу изменения в fstab :
```bash
LABEL=sandisk128 /mnt/data ext4 defaults 0 2
```

Монтирую файловую систему:
```bash
sudo mount -a
```

Проверяю что диск подключен:
```bash
df -h -x tmpfs
```

Создаю тестовый файл, проверяю его наличие и удаляю:
```bash
echo "success" | sudo tee /mnt/data/test_file
cat /mnt/data/test_file
sudo rm /mnt/data/test_file
```

Перезагружаю сервер и проверяю налииче монтированного диска:
```bash
sudo reboot now
lsblk
```

Меняю владельца на /mnt/data/ :
```bash
sudo chown -R postgres:postgres /mnt/data/
```

Переношу данные Postgres на монтированный образ:
```bash
sudo mv /var/lib/postgresql/18 /mnt/data
```

Пытаюсь запустить кластер:
```bash
sudo -u postgres pg_ctlcluster 18 main start
```

Не получается, **"Error: /var/lib/postgresql/18/main is not accessible or does not exist"**

Это связано с тем что я не указал новое расположение данных на монтированном томе, следовательно надо надо искать postgresql.conf:
'''bash
sudo nano /etc/postgresql/18/main/postgresql.conf
```

В файле нашел расположение "data_directory" и поменял путь на "/mnt/data/18/main", после запустил кластер командой:
```bash
sudo systemctl start postgresql@18-main
```

Подключаюсь к БД:
```bash
sudo -u postgres psql
```

Делаю выборку по ранее созданной таблице:
```pgsql
SELECT * FROM test1;
```

В ответ получил ранее созданную строку.
