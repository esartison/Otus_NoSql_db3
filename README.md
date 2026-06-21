# Домашнее задание Сартисона Евгения N3 #

Необходимо:

- построить шардированный кластер из 3 кластерных нод( по 3 инстанса с репликацией) и с кластером конфига(3 инстанса);
- добавить балансировку, нагрузить данными, выбрать хороший ключ шардирования, посмотреть как данные перебалансируются между шардами;
- поронять разные инстансы, посмотреть, что будет происходить, поднять обратно. Описать что произошло.
- настроить аутентификацию и многоролевой доступ;






## (1) описание подхода #

Используем подход по схеме, у нас 4 VM и на каждом из сервере бежить по одному мастер shard экземляра и 2 реплики других шардов
<img width="1100" height="1132" alt="image" src="https://github.com/user-attachments/assets/1427eab5-0a62-44b9-be9b-e3c2e03f1ef7" />

Развернул локально 4 виртуальные машины
```
192.168.0.20 mongodb1
192.168.0.21 mongodb2
192.168.0.22 mongodb3
192.168.0.23 mongosserver
```


## (2) подготовка VM для установки и установка MONGODB #

добавить список серверов в /etc/hosts
```
sudo tee -a /etc/hosts <<EOF
192.168.0.20 mongodb1
192.168.0.21 mongodb2
192.168.0.22 mongodb3
192.168.0.23 mongosserver
EOF
```
<img width="648" height="285" alt="image" src="https://github.com/user-attachments/assets/99ae7063-e33c-4c18-8a1e-1ca683b562ae" />

отключить firewall
>sudo ufw disable


добавить repo для mongodb и установка
>sudo apt-get install gnupg curl -y
>curl -fsSL https://pgp.mongodb.com/server-8.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor
>echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu $(lsb_release -cs)/mongodb-org/8.0 multiverse" | >sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
>sudo apt  -o Acquire::https::Verify-Peer=false -o Acquire::https::Verify-Host=false update
>sudo apt  -o Acquire::https::Verify-Peer=false -o Acquire::https::Verify-Host=false install -y mongodb-org

отключить автостарт MONGODB
>sudo systemctl status mongod
>sudo systemctl disable mongod
<img width="899" height="211" alt="image" src="https://github.com/user-attachments/assets/0c40a1c7-fc02-4daa-8230-15d144608ee9" />


Создать необходимые директории
```
sudo mkdir -p /data/mongodb/data/configsvr
sudo mkdir -p /data/mongodb/data/shardsvr-01
sudo mkdir -p /data/mongodb/data/shardsvr-02
sudo mkdir -p /data/mongodb/data/shardsvr-03

sudo mkdir -p /data/mongodb/log
```


## (3) настройка MongoDB(Config Server) ##

создать конфиг файл на серверах mongodb[1-3]
```
cat <<'EOF' > /etc/configsvr.conf
sharding:
   clusterRole: configsvr

replication:
   replSetName: configsvr

systemLog:
   destination: file
   path: /data/mongodb/log/configsvr.log
   logAppend: true

storage:
   dbPath: /data/mongodb/data/configsvr

net:
   bindIp: 0.0.0.0
   port: 27020

setParameter:
   enableLocalhostAuthBypass: false
EOF
```

создать сервис на серверах  mongodb[1-3]
```
cat <<'EOF' > /etc/systemd/system/configsvr.service
[Unit]
Description=MongoDB Config Server
After=network.target

[Service]
User=root
ExecStart=/usr/bin/mongod --config /etc/configsvr.conf
PIDFile=/var/run/mongodb/mongod.pid
Restart=on-failure
LimitNOFILE=64000
Type=simple

[Install]
WantedBy=multi-user.target
EOF
```
>systemctl daemon-reload
>systemctl enable configsvr.service
>systemctl start configsvr.service
>systemctl status configsvr.service
<img width="1893" height="252" alt="image" src="https://github.com/user-attachments/assets/064c8552-0459-410c-912b-a4580f3d36b2" />

на mongodb1 выполнить
>mongosh --port 27020
```
rs.initiate({
  _id: "configsvr",
  configsvr: true,
  members: [
    { _id: 0, host: "mongodb1:27020" },
    { _id: 1, host: "mongodb2:27020" },
    { _id: 2, host: "mongodb3:27020" },
  ],
});
```
<img width="710" height="327" alt="image" src="https://github.com/user-attachments/assets/3854b74b-d232-4ea5-84d7-1a03d5dd3096" />

Проверка статуса
```
rs.status().members.map(m => ({
  name: m.name,
  stateStr: m.stateStr,
  health: m.health
}))
```
<img width="681" height="220" alt="image" src="https://github.com/user-attachments/assets/fa6111ce-7bc2-4a0e-996b-1bf16b5079f4" />

Все хорошо!


## (4) настройка MongoDB(Shards) ##

Порты для шардов
```
shardsvr-01 → port 27021
shardsvr-02 → port 27022
shardsvr-03 → port 27023
```

#shardsvr-01#
создать конфиги на всех 3х серверах mongodb[1-3]
```
cat <<'EOF' > /etc/shardsvr-01.conf
storage:
  dbPath: /data/mongodb/data/shardsvr-01

systemLog:
  destination: file
  logAppend: true
  path: /data/mongodb/log/shardsvr-01.log

net:
  port: 27021
  bindIp: 0.0.0.0

replication:
  oplogSizeMB: 50
  replSetName: shardsvr-01

sharding:
  clusterRole: shardsvr
EOF


cat <<'EOF' > /etc/systemd/system/shardsvr-01.service
[Unit]
Description=MongoDB Database Server
Documentation=https://docs.mongodb.org/manual
After=network.target

[Service]
User=root
Group=root
ExecStart=/usr/bin/mongod --config /etc/shardsvr-01.conf
PIDFile=/var/run/mongodb/shardsvr-01.pid
# file size
LimitFSIZE=infinity
# cpu time
LimitCPU=infinity
# virtual memory size
LimitAS=infinity
# open files
LimitNOFILE=64000
# processes/threads
LimitNPROC=64000
# locked memory
LimitMEMLOCK=infinity
# total threads (user+kernel)
TasksMax=infinity
TasksAccounting=false

# Recommended limits for for mongod as specified in
# http://docs.mongodb.org/manual/reference/ulimit/#recommended-settings

[Install]
WantedBy=multi-user.target
EOF
```
перезапустить сервис
>systemctl daemon-reload
>systemctl enable shardsvr-01.service
>systemctl start shardsvr-01.service
>systemctl status shardsvr-01.service
<img width="925" height="231" alt="image" src="https://github.com/user-attachments/assets/97c70c7a-c796-4137-9d6e-16fe6f289e83" />


#shardsvr-02#
создать конфиги на всех 3х серверах mongodb[1-3]
```
cat <<'EOF' > /etc/shardsvr-02.conf
storage:
  dbPath: /data/mongodb/data/shardsvr-02

systemLog:
  destination: file
  logAppend: true
  path: /data/mongodb/log/shardsvr-02.log

net:
  port: 27022
  bindIp: 0.0.0.0

replication:
  oplogSizeMB: 50
  replSetName: shardsvr-02

sharding:
  clusterRole: shardsvr
EOF


cat <<'EOF' > /etc/systemd/system/shardsvr-02.service
[Unit]
Description=MongoDB Database Server
Documentation=https://docs.mongodb.org/manual
After=network.target

[Service]
User=root
Group=root
ExecStart=/usr/bin/mongod --config /etc/shardsvr-02.conf
PIDFile=/var/run/mongodb/shardsvr-02.pid
# file size
LimitFSIZE=infinity
# cpu time
LimitCPU=infinity
# virtual memory size
LimitAS=infinity
# open files
LimitNOFILE=64000
# processes/threads
LimitNPROC=64000
# locked memory
LimitMEMLOCK=infinity
# total threads (user+kernel)
TasksMax=infinity
TasksAccounting=false

# Recommended limits for for mongod as specified in
# http://docs.mongodb.org/manual/reference/ulimit/#recommended-settings

[Install]
WantedBy=multi-user.target
EOF
```

перезапустить сервис
>systemctl daemon-reload
>systemctl enable shardsvr-02.service
>systemctl start shardsvr-02.service
>systemctl status shardsvr-02.service
<img width="926" height="229" alt="image" src="https://github.com/user-attachments/assets/43614557-6b1c-42d4-85bf-739d3b690c39" />



#shardsvr-03#
создать конфиги на всех 3х серверах mongodb[1-3]
```
cat <<'EOF' > /etc/shardsvr-03.conf
storage:
  dbPath: /data/mongodb/data/shardsvr-03

systemLog:
  destination: file
  logAppend: true
  path: /data/mongodb/log/shardsvr-03.log

net:
  port: 27023
  bindIp: 0.0.0.0

replication:
  oplogSizeMB: 50
  replSetName: shardsvr-03

sharding:
  clusterRole: shardsvr
EOF


cat <<'EOF' > /etc/systemd/system/shardsvr-03.service
[Unit]
Description=MongoDB Database Server
Documentation=https://docs.mongodb.org/manual
After=network.target

[Service]
User=root
Group=root
ExecStart=/usr/bin/mongod --config /etc/shardsvr-03.conf
PIDFile=/var/run/mongodb/shardsvr-03.pid
# file size
LimitFSIZE=infinity
# cpu time
LimitCPU=infinity
# virtual memory size
LimitAS=infinity
# open files
LimitNOFILE=64000
# processes/threads
LimitNPROC=64000
# locked memory
LimitMEMLOCK=infinity
# total threads (user+kernel)
TasksMax=infinity
TasksAccounting=false

# Recommended limits for for mongod as specified in
# http://docs.mongodb.org/manual/reference/ulimit/#recommended-settings

[Install]
WantedBy=multi-user.target
EOF
```
<img width="1235" height="290" alt="image" src="https://github.com/user-attachments/assets/0b01a1c4-da87-4095-90c7-f1619bea8eb8" />


перезапустить сервис
>systemctl daemon-reload
>systemctl enable shardsvr-03.service
>systemctl start shardsvr-03.service
>systemctl status shardsvr-03.service


## (5) инициализировать реплики ##

#mongodb1# port 27021

>mongosh --port 27021

Инициализация реплик
```
rs.initiate({
  _id: "shardsvr-01",
  members: [
    { _id: 0, host: "mongodb1:27021" },
    { _id: 1, host: "mongodb2:27021" },
    { _id: 2, host: "mongodb3:27021" }
  ]
})
```

Првоерка статуса
```
rs.status().members.map(m => ({
   name: m.name,
   stateStr: m.stateStr,
   health: m.health
}))
```
<img width="727" height="210" alt="image" src="https://github.com/user-attachments/assets/2ee618d5-51b3-4c86-89a4-c4ebdb3ab18b" />




#mongodb2# port 27022

#mongodb3# port 27023

