# Урок "Настройка PostgreSQL"

Создаю базу 'otus':
```pgsql
CREATE DATABASE otus;
```

Захожу под пользователем postgres и запускаю pg_bench:
```bash
sudo su postgres
pgbench -i otus
```
В ответ получил:
```bash
generating data (client-side)...
100000 of 100000 tuples (100%) of pgbench_accounts done (elapsed 0.08 s, remaini                                                                                vacuuming...
creating primary keys...
done in 0.42 s (drop tables 0.01 s, create tables 0.02 s, client-side generate 0.29 s, vacuum 0.05 s, primary keys 0.06 s).
```

Тестирую производительность по дефолту:
```bash
pgbench -c 50 -j 2 -P 10 -T 60 otus
```

В ответ получаю ответ:
```bash
pgbench (18.0 (Ubuntu 18.0-1.pgdg22.04+3))
starting vacuum...end.
progress: 10.0 s, 413.9 tps, lat 115.598 ms stddev 141.389, 0 failed
progress: 20.0 s, 419.3 tps, lat 118.506 ms stddev 149.538, 0 failed
progress: 30.0 s, 429.3 tps, lat 117.607 ms stddev 161.997, 0 failed
progress: 40.0 s, 430.2 tps, lat 115.684 ms stddev 138.529, 0 failed
progress: 50.0 s, 410.3 tps, lat 121.738 ms stddev 154.182, 0 failed
progress: 60.0 s, 433.5 tps, lat 114.863 ms stddev 130.678, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 25415
number of failed transactions: 0 (0.000%)
latency average = 117.564 ms
latency stddev = 146.580 ms
initial connection time = 323.082 ms
tps = 424.779378 (without initial connection time)
```

Изменяю **max_worker_processes** с 8 на 4, так как через **cat /proc/cpuinfo | grep processor | wc -l
** у меня отображается 4 ядра:
```pgsql
ALTER SYSTEM SET max_worker_processes = 4;
```
Так как у меня **max_parallel_maintenance_workers** = 2 и **max_parallel_workers_per_gather** их я не трогаю и ребутаю кластер:
```bash

Запускаю тест производительности:
```bash
pgbench -c 50 -j 2 -P 10 -T 60 otus
```

В ответ получаю:
```bash
pgbench (18.0 (Ubuntu 18.0-1.pgdg22.04+3))
starting vacuum...end.
progress: 10.0 s, 406.4 tps, lat 116.973 ms stddev 132.070, 0 failed
progress: 20.0 s, 412.8 tps, lat 121.207 ms stddev 149.733, 0 failed
progress: 30.0 s, 431.0 tps, lat 116.603 ms stddev 146.515, 0 failed
progress: 40.0 s, 416.3 tps, lat 119.304 ms stddev 138.804, 0 failed
progress: 50.0 s, 414.0 tps, lat 120.932 ms stddev 153.294, 0 failed
progress: 60.0 s, 433.9 tps, lat 115.224 ms stddev 165.875, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 25194
number of failed transactions: 0 (0.000%)
latency average = 118.599 ms
latency stddev = 148.475 ms
initial connection time = 320.916 ms
tps = 421.084595 (without initial connection time)
```

Так как у меня 16ГБ оперативной памяти, вношу следующие   изменения в использование оперативной памяти:
```pgsql
ALTER SYSTEM SET shared_buffers = '6400MB';

ALTER SYSTEM SET effective_cache_size = '9000MB';
```
Перезагружаю кластер:
```bash
sudo systemctl restart postgresql@18-main
```

Захожу под **postgres** и запускаю тест производительности:
```bash
sudo su postgres && pgbench -c 50 -j 2 -P 10 -T 60 otus
```

В ответ получаю:
```bash
pgbench (18.0 (Ubuntu 18.0-1.pgdg22.04+3))
starting vacuum...end.
progress: 10.0 s, 388.6 tps, lat 122.709 ms stddev 159.961, 0 failed
progress: 20.0 s, 433.0 tps, lat 115.643 ms stddev 134.309, 0 failed
progress: 30.0 s, 410.1 tps, lat 121.433 ms stddev 140.315, 0 failed
progress: 40.0 s, 431.8 tps, lat 116.161 ms stddev 160.637, 0 failed
progress: 50.0 s, 426.0 tps, lat 117.304 ms stddev 141.298, 0 failed
progress: 60.0 s, 414.7 tps, lat 120.220 ms stddev 172.265, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 25091
number of failed transactions: 0 (0.000%)
latency average = 119.469 ms
latency stddev = 152.977 ms
initial connection time = 327.576 ms
tps = 417.461014 (without initial connection time)
```

Вношу дополнительные изменения в использование оперативной памяти:
```pgsql
ALTER SYSTEM SET temp_buffers = '4MB';

ALTER SYSTEM SET work_mem = '16MB';

ALTER SYSTEM SET maintenance_work_mem = '4GB';
```

Перезагружаю кластер:
```bash
sudo systemctl restart postgresql@18-main
```

Запускаю тест производительности:
```bash
pgbench -c 50 -j 2 -P 10 -T 60 otus
```


В ответ получаю:
```bash
pgbench (18.0 (Ubuntu 18.0-1.pgdg22.04+3))
starting vacuum...end.
progress: 10.0 s, 384.7 tps, lat 123.708 ms stddev 149.537, 0 failed
progress: 20.0 s, 414.2 tps, lat 120.878 ms stddev 148.748, 0 failed
progress: 30.0 s, 432.6 tps, lat 115.837 ms stddev 154.885, 0 failed
progress: 40.0 s, 424.7 tps, lat 117.593 ms stddev 144.090, 0 failed
progress: 50.0 s, 410.0 tps, lat 121.594 ms stddev 171.051, 0 failed
progress: 60.0 s, 432.5 tps, lat 115.164 ms stddev 151.746, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 25036
number of failed transactions: 0 (0.000%)
latency average = 119.316 ms
latency stddev = 154.098 ms
initial connection time = 332.950 ms
tps = 418.459562 (without initial connection time)
```

Вношу изменения в использоание жесткого диска:
```pgsql
ALTER SYSTEM SET fsync = 'off';

ALTER SYSTEM SET synchronous_commit = 'off';

ALTER SYSTEM SET random_page_cost = 2;

ALTER SYSTEM SET track_activity_query_size = '1MB';
```

Перезагружаю кластер:
```bash
sudo systemctl restart postgresql@18-main
```

Запускаю тест производительности:
```bash
pgbench -c 50 -j 2 -P 10 -T 60 otus
```

В ответ получаю:
```bash
pgbench (18.0 (Ubuntu 18.0-1.pgdg22.04+3))
starting vacuum...end.
progress: 10.0 s, 3619.5 tps, lat 13.332 ms stddev 16.806, 0 failed
progress: 20.0 s, 3785.8 tps, lat 13.211 ms stddev 17.172, 0 failed
progress: 30.0 s, 3803.6 tps, lat 13.132 ms stddev 16.876, 0 failed
progress: 40.0 s, 3832.0 tps, lat 13.061 ms stddev 16.755, 0 failed
progress: 50.0 s, 3883.8 tps, lat 12.871 ms stddev 16.271, 0 failed
progress: 60.0 s, 3887.0 tps, lat 12.858 ms stddev 15.686, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 228166
number of failed transactions: 0 (0.000%)
latency average = 13.080 ms
latency stddev = 16.605 ms
initial connection time = 326.766 ms
tps = 3820.510326 (without initial connection time)
```

## Вердикт
Разница с включенными показателями по дефолту и модернизированными составляет **3395.730948 TPS** это ОЧЕНЬ СИЛЬНЫЙ прирост производительности.
