# Домашнее задание Сартисона Евгения N3 #

Необходимо:

- построить шардированный кластер из 3 кластерных нод( по 3 инстанса с репликацией) и с кластером конфига(3 инстанса);
- добавить балансировку, нагрузить данными, выбрать хороший ключ шардирования, посмотреть как данные перебалансируются между шардами;
- поронять разные инстансы, посмотреть, что будет происходить, поднять обратно. Описать что произошло.
- настроить аутентификацию и многоролевой доступ;






## (1) описание подхода #

Используем подход по схеме, у нас 4 VM и на каждом из сервере бежить по одному мастер shard экземляра и 2 реплики других шардов
<img width="1100" height="1132" alt="image" src="https://github.com/user-attachments/assets/2f9d9c94-ee93-475d-ad6d-ab439c09e1e3" />

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

# shardsvr-01 #

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


# shardsvr-02 #

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



# shardsvr-03 #

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

# mongodb1 # port 27021

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




# mongodb2 # port 27022

>mongosh --port 27022

Инициализация реплик
```
rs.initiate({
  _id: "shardsvr-02",
  members: [
    { _id: 0, host: "mongodb1:27022" },
    { _id: 1, host: "mongodb2:27022" },
    { _id: 2, host: "mongodb3:27022" }
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
<img width="715" height="207" alt="image" src="https://github.com/user-attachments/assets/2be3f81a-1ca8-47fb-b052-07495589fd9e" />


# mongodb3 # port 27023
#mongodb3# port 27023
>mongosh --port 27023

Инициализация реплик
```
rs.initiate({
  _id: "shardsvr-03",
  members: [
    { _id: 0, host: "mongodb1:27023" },
    { _id: 1, host: "mongodb2:27023" },
    { _id: 2, host: "mongodb3:27023" }
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
<img width="713" height="212" alt="image" src="https://github.com/user-attachments/assets/9e641755-5245-4d17-8065-e7d4a9d22097" />


## (5) конфигурация MONGOS ##

На сервере mongosserver coздать конфиг файл
```
cat <<'EOF' > /etc/mongos.conf
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongos.log

net:
  port: 27017
  bindIp: 0.0.0.0

sharding:
  configDB: configsvr/mongodb1:27020,mongodb2:27020,mongodb3:27020
EOF
```


создать systemd service:
```
cat <<'EOF' > /etc/systemd/system/mongos.service
[Unit]
Description=MongoDB Database Server
Documentation=https://docs.mongodb.org/manual
After=network.target

[Service]
User=root
Group=root
ExecStart=/usr/bin/mongos --config /etc/mongos.conf
PIDFile=/var/run/mongodb/mongos.pid
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

Запустить службу

>systemctl daemon-reload

>systemctl enable  mongos.service

>systemctl start  mongos.service

>systemctl status  mongos.service
<img width="755" height="187" alt="image" src="https://github.com/user-attachments/assets/1e67bab7-f90a-4355-806b-beb191f7035c" />


добавить шарды в кластер на сервере mongosserver
> mongosh --port 27017

>sh.addShard("shardsvr-01/mongodb1:27021")
>sh.addShard("shardsvr-02/mongodb2:27022")
>sh.addShard("shardsvr-03/mongodb3:27023")

>sh.status()
```
[direct: mongos] test> sh.status()
shardingVersion
{ _id: 1, clusterId: ObjectId('6a3810670b32382b37e4417a') }
---
shards
[
  {
    _id: 'shardsvr-01',
    host: 'shardsvr-01/mongodb1:27021,mongodb2:27021,mongodb3:27021',
    state: 1,
    topologyTime: Timestamp({ t: 1782063683, i: 4 }),
    replSetConfigVersion: Long('-1')
  },
  {
    _id: 'shardsvr-02',
    host: 'shardsvr-02/mongodb2:27022,mongodb1:27022,mongodb3:27022',
    state: 1,
    topologyTime: Timestamp({ t: 1782063693, i: 9 }),
    replSetConfigVersion: Long('-1')
  },
  {
    _id: 'shardsvr-03',
    host: 'shardsvr-03/mongodb3:27023,mongodb1:27023,mongodb2:27023',
    state: 1,
    topologyTime: Timestamp({ t: 1782063702, i: 9 }),
    replSetConfigVersion: Long('-1')
  }
]
---
active mongoses
[ { '8.0.26': 1 } ]
---
autosplit
{ 'Currently enabled': 'yes' }
---
balancer
{
  'Currently enabled': 'yes',
  'Failed balancer rounds in last 5 attempts': 0,
  'Currently running': 'no',
  'Migration Results for the last 24 hours': 'No recent migrations'
}
---
shardedDataDistribution
[]
---
databases
[
  {
    database: { _id: 'config', primary: 'config', partitioned: true },
    collections: {}
  }
]
```


## (6) выставить приоритет шардам на сервере mongosserver ##

# shardsvr-01 #
на сервере mongodb1

подключиться к shardsvr-01
mongosh --port 27021
> rs.conf().members.map(m => ({ _id: m._id, host: m.host }))
<img width="1170" height="136" alt="image" src="https://github.com/user-attachments/assets/14489803-ac6e-45e1-a3d8-b138e92cdda7" />

настроить mongodb1 как главный узел
>cfg = rs.conf()
>cfg.members[0].priority = 2   // primary priority
>cfg.members[1].priority = 1
>cfg.members[2].priority = 1
>rs.reconfig(cfg)
>rs.stepDown()
>rs.conf().members.forEach(m => print(m.host + " → priority: " + m.priority))
<img width="1147" height="85" alt="image" src="https://github.com/user-attachments/assets/8d597e06-165d-4433-a652-fde98b504ce6" />




# shardsvr-02 #
на сервере mongodb2

подключиться к shardsvr-02
mongosh --port 27022
> rs.conf().members.map(m => ({ _id: m._id, host: m.host }))
<img width="952" height="140" alt="image" src="https://github.com/user-attachments/assets/b5808c40-8858-4aab-9ebf-58ad8efd12fc" />

>rs.conf().members.forEach(m => print(m.host + " → priority: " + m.priority))
<img width="1122" height="89" alt="image" src="https://github.com/user-attachments/assets/8d3a0d63-6dee-4e09-b4e5-42e2c404ce0c" />

настроить mongodb2 как главный узел
>cfg = rs.conf()
>cfg.members[0].priority = 1   // primary priority
>cfg.members[1].priority = 2
>cfg.members[2].priority = 1
>rs.reconfig(cfg)
>rs.stepDown()
>rs.conf().members.forEach(m => print(m.host + " → priority: " + m.priority))
<img width="1151" height="76" alt="image" src="https://github.com/user-attachments/assets/1f63dd80-3e39-40d8-ab00-f1e5aa9c769b" />



# shardsvr-03 #
на сервере mongodb3

подключиться к shardsvr-03
mongosh --port 27023
> rs.conf().members.map(m => ({ _id: m._id, host: m.host }))
<img width="939" height="141" alt="image" src="https://github.com/user-attachments/assets/353255a0-f34e-46d9-8bf9-cb432fb7513b" />

>rs.conf().members.forEach(m => print(m.host + " → priority: " + m.priority))
<img width="1145" height="116" alt="image" src="https://github.com/user-attachments/assets/25117c24-9820-41e7-bc7c-8e4c5938845d" />

настроить mongodb3 как главный узел
>cfg = rs.conf()
>cfg.members[0].priority = 1   // primary priority
>cfg.members[1].priority = 1
>cfg.members[2].priority = 2
>rs.reconfig(cfg)
>rs.stepDown()
>rs.conf().members.forEach(m => print(m.host + " → priority: " + m.priority))
<img width="1158" height="90" alt="image" src="https://github.com/user-attachments/assets/165f4995-557b-44ae-81b8-92b2dfe1dbc7" />

## (7) создание базы и настройка шардирования для коллекции ##

с сервера mongosserver подключиться к базе
>mongosh --port 27017

создать базы sales и коллекцию orders
>use sales
>db.createCollection("orders")

включение шардирования на уровне базы
>sh.enableSharding("sales")
<img width="709" height="253" alt="image" src="https://github.com/user-attachments/assets/f5653ec0-1d74-487b-ac9b-58698516efcf" />

включение шардирования на уровне коллекции по полю orderid и алгоритмом HASH
>sh.shardCollection("sales.orders", { orderId: "hashed" });
<img width="848" height="269" alt="image" src="https://github.com/user-attachments/assets/73cb1c64-6ca4-4667-a47c-15e520ffd0df" />

сделать тестовую вставку в шардированную коллекцию
```
let bulk = [];
const totalDocs = 1_000_000;     // total number of documents to insert
const batchSize = 20_000;        // larger batch size reduces round-trips

for (let i = 0; i < totalDocs; i++) {
  bulk.push({
    orderId: i,
    orderDate: new Date(2000 + (i % 20), 0, 1),
    amount: Math.floor(Math.random() * 1000)
  });

  if (bulk.length === batchSize) {
    db.orders.insertMany(bulk, { ordered: false });  // unordered insert
    bulk = [];
  }
}

// Insert the remaining documents
if (bulk.length > 0) {
  db.orders.insertMany(bulk, { ordered: false });
}

print(`✅ Successfully inserted ${totalDocs} documents into sales.orders`);
```
<img width="1007" height="81" alt="image" src="https://github.com/user-attachments/assets/5c60d4d4-0ee2-4f29-b377-36d3dc4e046e" />


Проверрить, была ли таблица вообще шардирована
>use config
>db.collections.find({ _id: "sales.orders" }).pretty()
<img width="855" height="309" alt="image" src="https://github.com/user-attachments/assets/d4a71a19-afba-4079-a136-56651a8022a5" />

Проверить как данные распределены между шардами
> use sales
> db.orders.getShardDistribution()
```
[direct: mongos] sales> db.orders.getShardDistribution()
Shard shardsvr-02 at shardsvr-02/mongodb1:27022,mongodb2:27022,mongodb3:27022
{
  data: '21.01MiB',
  docs: 333819,
  chunks: 1,
  'estimated data per chunk': '21.01MiB',
  'estimated docs per chunk': 333819
}
---
Shard shardsvr-03 at shardsvr-03/mongodb1:27023,mongodb2:27023,mongodb3:27023
{
  data: '21.01MiB',
  docs: 333951,
  chunks: 1,
  'estimated data per chunk': '21.01MiB',
  'estimated docs per chunk': 333951
}
---
Shard shardsvr-01 at shardsvr-01/mongodb1:27021,mongodb2:27021,mongodb3:27021
{
  data: '20.91MiB',
  docs: 332230,
  chunks: 1,
  'estimated data per chunk': '20.91MiB',
  'estimated docs per chunk': 332230
}
---
Totals
{
  data: '62.94MiB',
  docs: 1000000,
  chunks: 3,
  'Shard shardsvr-02': [
    '33.38 % data',
    '33.38 % docs in cluster',
    '66B avg obj size on shard'
  ],
  'Shard shardsvr-03': [
    '33.39 % data',
    '33.39 % docs in cluster',
    '66B avg obj size on shard'
  ],
  'Shard shardsvr-01': [
    '33.22 % data',
    '33.22 % docs in cluster',
    '66B avg obj size on shard'
  ]
}
[direct: mongos] sales> 
```
Шардирование работает, данные распределены между шардами.


## (8) поронять разные инстансы, посмотреть, что будет происходить, поднять обратно. Описать что произошло ##

## (9) настроить аутентификацию и многоролевой доступ ##

https://maihoangviet.medium.com/hardening-an-existing-mongodb-sharded-cluster-with-keyfile-authentication-20944afd13b1
