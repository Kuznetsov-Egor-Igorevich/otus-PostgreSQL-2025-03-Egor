# Проект "Создание и тестирование высоконагруженного отказоустойчивого кластера PostgreSQL на базе Patroni"

## Введение
Есть пять машин:
1. HP Elite Desk  - Intel(R) Core(TM) i5-6500 CPU @ 3.20GHz, 16 Gb RAM, Ubuntu 24.04, XXX.XXX.**48.82**
2. Raspberry Pi 5 - ARM Cortex-A76 4x-2400 MHz, 8 Gb RAM, Ubuntu 24.04, XXX.XXX.**200.33**
3. Raspberry Pi 3 - ARM Cortex-A53 4x-1200 MHz, 1 Gb RAM, Ubuntu 24.04, XXX.XXX.**213.117**
4. Raspberry Pi zero 2w - ARM Cortex-A53 4x-1000 Mhz, 512 Mb RAM, Ubuntu 24.04, XXX.XXX.**206.86**
5. MacBook - Apple M4 10x-2400 MHz, 16 Gb RAM, Mac Os 26.2, XXX.XXX.**9.84**

## Установка ETCD

**Далее действия будут выполняться на трех машинах, в порядке очередности перечисленной во введении**

Устанавливаю etcd:
```bash
sudo apt -y install etcd-server
sudo apt -y install etcd-client
```

Останавливаю etcd:
```bash
sudo systemctl stop etcd
sudo systemctl disable etcd
```

Настраиваю файл конфинурации:
```bash
sudo rm -rf /var/lib/etcd/default

sudo nano /etc/default/etcd

ETCD_NAME="etcd-OTUS-1"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://XXX.XXX.48.82:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://XXX.XXX.48.82:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd_OTUS_Claster"
ETCD_INITIAL_CLUSTER="etcd-OTUS-1=http://XXX.XXX.48.82:2380,etcd-OTUS-2=http://XXX.XXX.200.33:2380,etcd-OTUS-3=
http://XXX.XXX.213.117:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"

sudo -s

sudo cat > /etc/systemd/system/etcd.service

[Unit]
Description=etcd - highly-available key value store
Documentation=https://github.com/etcd-io/etcd
After=network.target
Wants=network-online.target

[Service]
Type=notify
User=etcd
Group=etcd
EnvironmentFile=-/etc/default/etcd
ExecStart=/usr/bin/etcd
Restart=on-failure
RestartSec=10s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

sudo chown -R etcd:etcd /var/lib/etcd
```


Настраиваю автозапуск etcd:
```bash
sudo systemctl daemon-reload

sudo systemctl enable etcd

sudo systemctl start etcd 
```

Проверяю работу кластера:
```bash
etcdctl endpoint status --cluster -w table
```

В ответ получаю:
```text
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|          ENDPOINT           |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|   http://XXX.93.200.33:2379 |  73e8f6e44b4f580 |  3.4.30 |   20 kB |     false |      false |       140 |          8 |                  8 |        |
| http://XXX.XXX.213.117:2379 | 51a02ba4fa68a159 |  3.4.30 |   20 kB |     false |      false |       140 |          8 |                  8 |        |
|   http://XXX.XXX.48.82:2379 | e97c07e98e3956d6 |  3.4.30 |   20 kB |      true |      false |       140 |          8 |                  8 |        |
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

## Настройка postgresql на первой машине XXX.XXX.48.82

Создаю нового пользователя:

```bash
CREATE USER replicator REPLICATION LOGIN ENCRYPTED PASSWORD 'OTUS2025';
```

Редактирую **pg_hba.conf**:

```bash
sudo nano /etc/postgresql/18/main/pg_hba.conf

...
host    all             all             XXX.XXX.200.33/XX       password
host    replication     replicator      XXX.XXX.200.33/XX       password
host    all             all             XXX.XXX.213.117/XX      password
host    replication     replicator      XXX.XXX.213.117/XX      password
...
```

Редактирую **postgresql.conf**: 
```bash
sudo nano /etc/postgresql/18/main/postgresql.conf 

...
listen_addresses = 'XXX.XXX.48.82'
...

sudo systemctl restart postgresql
```

Подготавливаю вторую и третью машину:

```bash
sudo systemctl stop postgresql

sudo rm -rf /var/lib/postgresql/18/main/*
```

## Настройка **Patroni**

**Далее действия будут выполняться на трех машинах, в порядке очередности перечисленной во введении**

Установка patroni:
```bash
sudo apt update && sudo apt install -y python3 python3-pip

sudo mkdir -p /opt/patroni

sudo python3 -m venv /opt/patroni/venv

sudo apt install python3.12-venv

sudo /opt/patroni/venv/bin/pip install 'patroni[etcd]'

sudo chown postgres:postgres /opt/patroni

sudo  /opt/patroni/venv/bin/pip install 'psycopg2-binary'

sudo mkdir -p /data/patroni

sudo chown -R postgres:postgres /data/patroni

sudo chmod 700 /data/patroni

```

Настройка конфиг файла:
```bash
sudo nano /etc/patroni.yml

scope: otus_cluster
namespace: /patroni/
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: XXX.XXX.48.82:8008


etcd3:
  hosts:
    - XXX.XXX.48.82:2379
    - XXX.XXX.200.33:2379
    - XXX.XXX.213.117:2379


bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
  initdb:
    - encoding: UTF8
  pg_hba:
    - host replication replicator XXX.XX.200.33/10 password
    - host replication replicator XXX.XXX.213.117/10 password
    - host all all 0.0.0.0/0 password
  users:
    admin:
      password: ‘XXX’
      options:
        - createrole
        - createdb

postgresql:
  listen: 0.0.0.0:5432
  connect_address: XXX.XXX.48.82:5432
  data_dir: /data/patroni
  authentication:
    replication:
      username: replicator
      password: ‘XXX’
    superuser:
      username: postgres
      password: ‘XXX’


sudo nano /etc/systemd/system/patroni.service

[Unit]
Description=High availability PostgreSQL Cluster - Patroni
After=syslog.target network.target

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/opt/patroni/venv/bin/patroni /etc/patroni.yml
KillMode=process
TimeoutSec=30
Restart=no

[Install]
WantedBy=multi-user.target
```

Запуск patroni:
```bash
sudo systemctl daemon-reload

sudo systemctl enable patroni

sudo systemctl start patroni
```

Проверяю работу:
```bash
/opt/patroni/venv/bin/patronictl -c /etc/patroni.yml list
```

В ответ получаю:
```bash
+ Cluster: otus_cluster (7589614155091801433) ---+----+-------------+-----+------------+-----+
| Member | Host            | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+--------+-----------------+---------+-----------+----+-------------+-----+------------+-----+
| node1  | XXX.XXX.48.82   | Leader  | running   |  3 |             |     |            |     |
| node2  | XXX.XXX.200.33  | Replica | streaming |  3 |   0/5000168 |   0 |  0/5000168 |   0 |
| node3  | XXX.XXX.213.117 | Replica | streaming |  3 |   0/5000168 |   0 |  0/5000168 |   0 |
+--------+-----------------+---------+-----------+----+-------------+-----+------------+-----+
```

Проверяю работу, на первой машине остановлю patroni:
```bash
sudo systemctl stop patroni
```

Проверяю нового лидера:
```bash
/opt/patroni/venv/bin/patronictl -c /etc/patroni.yml list
```

В ответ получаю:
```bash
+ Cluster: otus_cluster (7589614155091801433) ---+----+-------------+-----+------------+-----+
| Member | Host            | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+--------+-----------------+---------+-----------+----+-------------+-----+------------+-----+
| node1  | XXX.XXX.48.82   | Replica | stopped   |    |     unknown |     |    unknown |     |
| node2  | XXX.XXX.200.33  | Leader  | running   |  4 |             |     |            |     |
| node3  | XXX.XXX.213.117 | Replica | streaming |  4 |   0/5018AD0 |   0 |  0/5018AD0 |   0 |
+--------+-----------------+---------+-----------+----+-------------+-----+------------+-----+
```

## Установка **Prometheus**

**Установка будет осуществляться на 4ую машину XXX.XXX.206.86**

Подготовка:
```bash
sudo useradd --no-create-home --shell /bin/false prometheus

sudo mkdir /etc/prometheus /var/lib/prometheus

sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus

wget https://github.com/prometheus/prometheus/releases/download/v2.47.0/prometheus-2.47.0.linux-arm64.tar.gz

tar xvf prometheus-2.47.0.linux-arm64.tar.gz

cd prometheus-2.47.0.linux-arm64/

sudo cp prometheus /usr/bin/

sudo cp promtool /usr/bin/

sudo chown prometheus:prometheus /usr/bin/prometheus

sudo chown prometheus:prometheus /usr/bin/promtool
```

Настраиваю конфиг файл:
```bash
sudo nano /etc/prometheus/prometheus.yml

global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'patroni'
    static_configs:
      - targets:
        - 'XXX.XXX.48.82:8008'
        - 'XXX.XXX.200.33:8008'
        - 'XXX.XXX.213.117:8008'
    metrics_path: '/metrics'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance


  - job_name: 'postgres'
    static_configs:
      - targets:
        - 'XXX.XXX.48.82:9187'
        - 'XXX.XXX.200.33:9187'
        - 'XXX.XXX.213.117:9187'


  - job_name: 'node'
    static_configs:
      - targets:
        - 'XXX.XXX.48.82:9100'
        - 'XXX.XXX.200.33:9100'
        - 'XXX.XXX.213.117:9100'
        - 'XXX.XXX.206.86:9100'

sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml

sudo nano /etc/systemd/system/prometheus.service

[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries \
    --web.listen-address=0.0.0.0:9090
Restart=always

[Install]
WantedBy=multi-user.target
```

Запускаю prometheus:
```bash
sudo systemctl daemon-reload

sudo systemctl enable prometheus

sudo systemctl start prometheus

sudo systemctl status prometheus
```



## Установка **Node-Exporter**:

**Установка будет осуществляется на первые четыре машины**

Подготовка:
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz

tar xvf node_exporter-*.tar.gz

cd node_exporter-1.7.0.linux-amd64/

sudo cp node_exporter /usr/bin/

sudo useradd --no-create-home --shell /bin/false node_exporter

sudo chown node_exporter:node_exporter /usr/bin/node_exporter

sudo tee /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF


sudo systemctl daemon-reload

sudo systemctl enable node_exporter

sudo systemctl start node_exporter
```


## Установка **Postgres Exporter**

**Установка будет осуществляется на первые три машины**

Подготовка:
```bash
sudo apt install prometheus-postgres-exporter

sudo nano /etc/default/prometheus-postgres-exporter

postgresql://postgres:XXX!@XXX.XXX.213.117:5432/postgres?sslmode=disable

sudo systemctl enable prometheus-postgres-exporter

sudo systemctl start prometheus-postgres-exporter
```

## Установка **Grafana**

**Установка будет осуществляется на пятую машину**

```bash
brew install grafana

brew services start grafana
```
