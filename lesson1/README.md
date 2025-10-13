# Урок "SQL и реляционные СУБД. Введение в PostgreSQL"

## Введение

В рамках цикла обучения я буду использовать следующие инструменты:
 * Raspberry pi zero 2w;
 * Raspberry pi3;
 * Raspberry pi5;
 * Ubuntu 22.04.5 LTS в качестве основного сервера;
 * Собственно настроенный ВПН.

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
