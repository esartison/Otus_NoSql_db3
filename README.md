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


## (3) настройка MongoDB ##
Устанавливаем MongoDB на локальную VM и используем [Install MongoDB Community Edition on Ubuntu](https://www.mongodb.com/docs/v8.0/tutorial/install-mongodb-on-ubuntu/#install-mongodb-community-edition-on-ubuntu)


установка утилит gnupg и curl
>sudo apt-get install gnupg curl

импортировать публичный ключ
>curl -fsSL --insecure https://pgp.mongodb.com/server-8.0.asc |    sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg    --dearmor

создать список файлов
> echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/8.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

обновить базу
> sudo apt-get update -o Acquire::https::Verify-Peer=false
<img width="1037" height="265" alt="image" src="https://github.com/user-attachments/assets/74caba5f-2f92-45c5-9127-6f9eb6e15022" />

установка MongoDB 
> sudo apt-get install -y mongodb-org -o Acquire::https::Verify-Peer=false
<img width="951" height="583" alt="image" src="https://github.com/user-attachments/assets/6ad39f00-57fb-40a6-a30f-558e1292b1e7" />

старт MongoDB
>sudo systemctl start mongod
<img width="1883" height="292" alt="image" src="https://github.com/user-attachments/assets/6fb6c041-b3c2-4438-bf1c-1cc6ff306648" />

проверка подключения
<img width="1789" height="472" alt="image" src="https://github.com/user-attachments/assets/a15c7e16-5ff9-429f-9863-426df8f7d67e" />




## (2) заполнить данными ##
Использую данные из [Load Sample Data into MongoDB](https://restheart.org/docs/mongodb-rest/sample-data)

скачать data set
>curl --insecure https://atlas-education.s3.amazonaws.com/sampledata.archive -o sampledata.archive

сделать импорт баз
>mongorestore --archive=sampledata.archive
<img width="1882" height="234" alt="image" src="https://github.com/user-attachments/assets/ca5705b4-45d3-489c-913a-74a8da27726a" />

проверить базы данных

<img width="519" height="269" alt="image" src="https://github.com/user-attachments/assets/a0da79ba-91c4-46d2-add1-966edb867a81" />

подключиться к базе sample_restaurants

<img width="344" height="51" alt="image" src="https://github.com/user-attachments/assets/429f7d98-a26d-439e-a34b-87ead440d22d" />

проверка доступных коллекций

<img width="399" height="120" alt="image" src="https://github.com/user-attachments/assets/540269df-9dd3-42da-bfd5-eeff875ae66b" />

проверить размер загруженных коллекций
```
db = db.getSiblingDB("admin");
const dbs = db.adminCommand("listDatabases").databases;

print("Database, Collection, DataSize (MB), StorageSize (MB)");

dbs.forEach(function(d) {
    // Skip system databases to clear up clutter
    if (["admin", "config", "local"].includes(d.name)) return;
    
    let currentDb = db.getSiblingDB(d.name);
    let collections = currentDb.getCollectionNames();
    
    collections.forEach(function(c) {
        let stats = currentDb.getCollection(c).stats();
        
        // Convert bytes to Megabytes for readability
        let dataSizeMb = (stats.size / (1024 * 1024)).toFixed(2);
        let storageSizeMb = (stats.storageSize / (1024 * 1024)).toFixed(2);
        
        print(`${d.name}, ${c}, ${dataSizeMb} MB, ${storageSizeMb} MB`);
    });
});
```
<img width="607" height="455" alt="image" src="https://github.com/user-attachments/assets/b3d0eeee-e160-496c-bdaf-33a9f32ba946" />


## (3) написать несколько запросов на выборку и обновление данных ##

Сделать выборку всего из таблицы
>sample_restaurants> db.getCollection('restaurants').find({})
<img width="831" height="704" alt="image" src="https://github.com/user-attachments/assets/f6898d25-c03e-45f4-9366-fcacf7d72be4" />

Сделать выборку и вывести только 3 поля
>sample_restaurants> db.getCollection('restaurants').find({}, { restaurant_id: 1, name: 1, address: 1 })
<img width="661" height="622" alt="image" src="https://github.com/user-attachments/assets/50b5a36d-36b2-40ef-86c9-7c1fa7dda531" />

вернуть поля в определенном порядке и не включать поле _ID
```
db.restaurants.aggregate([
  {
    $project: {
      _id: 0,
      restaurant_id: "$restaurant_id",    
      name: "$name",      
      address: "$address"       
    }
  }
])
```
<img width="596" height="385" alt="image" src="https://github.com/user-attachments/assets/9937778f-e177-4e81-b7e6-cff82c098857" />

Допустим ситуацию, сеть рестораном в том же здании решили открыть еще один ресторан русской кухни
```
-- Получаем информацию о уже существуюещем
sample_restaurants> db.getCollection('restaurants').find({restaurant_id: '40361708' })
[
  {
    _id: ObjectId('5eb3d668b31de5d588f4293d'),
    address: {
      building: '759',
      coord: [ -73.9925306, 40.7309346 ],
      street: 'Broadway',
      zipcode: '10003'
    },
    borough: 'Manhattan',
    cuisine: 'Delicatessen',
    grades: [
      { date: ISODate('2014-01-21T00:00:00.000Z'), grade: 'A', score: 12 },
      { date: ISODate('2013-01-04T00:00:00.000Z'), grade: 'A', score: 11 },
      { date: ISODate('2012-06-07T00:00:00.000Z'), grade: 'A', score: 6 },
      { date: ISODate('2012-01-17T00:00:00.000Z'), grade: 'A', score: 8 }
    ],
    name: "Bully'S Deli",
    restaurant_id: '40361708'
  }
]

-- заполняем переменную
var newRow =db.getCollection('restaurants').findOne({restaurant_id: '40361708' });

-- делаем нужные правки
newRow._id = new ObjectId();
newRow.cuisine = "Russian Delicatessen";
newRow.name =  "Bully'S Deli RU";
newRow.restaurant_id =  "4036444844";

-- Вставляем новую запись
db.restaurants.insertOne(newRow);
{
  acknowledged: true,
  insertedId: ObjectId('6a36c4a19188f806cf9df8a3')
}

-- проверяем
sample_restaurants> db.getCollection('restaurants').findOne({restaurant_id: '4036444844' });
{
  _id: ObjectId('6a36c4a19188f806cf9df8a3'),
  address: {
    building: '759',
    coord: [ -73.9925306, 40.7309346 ],
    street: 'Broadway',
    zipcode: '10003'
  },
  borough: 'Manhattan',
  cuisine: 'Russian Delicatessen',
  grades: [
    { date: ISODate('2014-01-21T00:00:00.000Z'), grade: 'A', score: 12 },
    { date: ISODate('2013-01-04T00:00:00.000Z'), grade: 'A', score: 11 },
    { date: ISODate('2012-06-07T00:00:00.000Z'), grade: 'A', score: 6 },
    { date: ISODate('2012-01-17T00:00:00.000Z'), grade: 'A', score: 8 }
  ],
  name: "Bully'S Deli RU",
  restaurant_id: '4036444844'
}
```

Потом решили провести ребрендинг и переименовать ресторан "Bully'S Deli RU" -> "BD RU VIP"
```
sample_restaurants> db.getCollection('restaurants').findOne({restaurant_id: "4036444844" });
{
  _id: ObjectId('6a36c4a19188f806cf9df8a3'),
  address: {
    building: '759',
    coord: [ -73.9925306, 40.7309346 ],
    street: 'Broadway',
    zipcode: '10003'
  },
  borough: 'Manhattan',
  cuisine: 'Russian Delicatessen',
  grades: [
    { date: ISODate('2014-01-21T00:00:00.000Z'), grade: 'A', score: 12 },
    { date: ISODate('2013-01-04T00:00:00.000Z'), grade: 'A', score: 11 },
    { date: ISODate('2012-06-07T00:00:00.000Z'), grade: 'A', score: 6 },
    { date: ISODate('2012-01-17T00:00:00.000Z'), grade: 'A', score: 8 }
  ],
  name: "Bully'S Deli RU",
  restaurant_id: '4036444844'
}

sample_restaurants> db.restaurants.updateOne( { restaurant_id: "4036444844" }, { $set: { name: "BD RU VIP" } } );
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 1,
  upsertedCount: 0
}

sample_restaurants> db.getCollection('restaurants').findOne({restaurant_id: "4036444844" });
{
  _id: ObjectId('6a36c4a19188f806cf9df8a3'),
  address: {
    building: '759',
    coord: [ -73.9925306, 40.7309346 ],
    street: 'Broadway',
    zipcode: '10003'
  },
  borough: 'Manhattan',
  cuisine: 'Russian Delicatessen',
  grades: [
    { date: ISODate('2014-01-21T00:00:00.000Z'), grade: 'A', score: 12 },
    { date: ISODate('2013-01-04T00:00:00.000Z'), grade: 'A', score: 11 },
    { date: ISODate('2012-06-07T00:00:00.000Z'), grade: 'A', score: 6 },
    { date: ISODate('2012-01-17T00:00:00.000Z'), grade: 'A', score: 8 }
  ],
  name: 'BD RU VIP',
  restaurant_id: '4036444844'
}
```



## (4) создать индексы и сравнить производительность* ##

Попробвал создать индекс для коллекции с размером 10Mb. После создания индекса время выполнения(executionTimeMillis) сократилось с 342 до 22.

Статистика ДО
```
sample_restaurants>  db.getCollection('restaurants').find({restaurant_id: '40361708' }).explain("executionStats")
{
  explainVersion: '1',
  queryPlanner: {
    namespace: 'sample_restaurants.restaurants',
    parsedQuery: { restaurant_id: { '$eq': '40361708' } },
    indexFilterSet: false,
    queryHash: '9687E8EA',
    planCacheShapeHash: '9687E8EA',
    planCacheKey: 'F4787E42',
    optimizationTimeMillis: 29,
    maxIndexedOrSolutionsReached: false,
    maxIndexedAndSolutionsReached: false,
    maxScansToExplodeReached: false,
    prunedSimilarIndexes: false,
    winningPlan: {
      isCached: false,
      stage: 'COLLSCAN',
      filter: { restaurant_id: { '$eq': '40361708' } },
      direction: 'forward'
    },
    rejectedPlans: []
  },
  executionStats: {
    executionSuccess: true,
    nReturned: 1,
    executionTimeMillis: 342,
    totalKeysExamined: 0,
    totalDocsExamined: 25359,
    executionStages: {
      isCached: false,
      stage: 'COLLSCAN',
      filter: { restaurant_id: { '$eq': '40361708' } },
      nReturned: 1,
      executionTimeMillisEstimate: 287,
      works: 25360,
      advanced: 1,
      needTime: 25358,
      needYield: 0,
      saveState: 14,
      restoreState: 14,
      isEOF: 1,
      direction: 'forward',
      docsExamined: 25359
    }
  },
  queryShapeHash: 'BB4853BE7BAEE5AC49F54556E5F106E34AE79411503664D5189653C291017885',
  command: {
    find: 'restaurants',
    filter: { restaurant_id: '40361708' },
    '$db': 'sample_restaurants'
  },
  serverInfo: {
    host: 'course',
    port: 27017,
    version: '8.0.26',
    gitVersion: 'a2c24805265db119cc39f2f54e067b4294a4ddd2'
  },
  serverParameters: {
    internalQueryFacetBufferSizeBytes: 104857600,
    internalQueryFacetMaxOutputDocSizeBytes: 104857600,
    internalLookupStageIntermediateDocumentMaxSizeBytes: 104857600,
    internalDocumentSourceGroupMaxMemoryBytes: 104857600,
    internalQueryMaxBlockingSortMemoryUsageBytes: 104857600,
    internalQueryProhibitBlockingMergeOnMongoS: 0,
    internalQueryMaxAddToSetBytes: 104857600,
    internalDocumentSourceSetWindowFieldsMaxMemoryBytes: 104857600,
    internalQueryFrameworkControl: 'trySbeRestricted',
    internalQueryPlannerIgnoreIndexWithCollationForRegex: 1
  },
  ok: 1
}
```

Создание индекса
```
sample_restaurants> db.users.createIndex( { "restaurant_id": 1 } )
restaurant_id_1
```

Статистика ПОСЛЕ
```
sample_restaurants> db.getCollection('restaurants').find({restaurant_id: '40361708' }).explain("executionStats")
{
  explainVersion: '1',
  queryPlanner: {
    namespace: 'sample_restaurants.restaurants',
    parsedQuery: { restaurant_id: { '$eq': '40361708' } },
    indexFilterSet: false,
    queryHash: '9687E8EA',
    planCacheShapeHash: '9687E8EA',
    planCacheKey: 'F4787E42',
    optimizationTimeMillis: 3,
    maxIndexedOrSolutionsReached: false,
    maxIndexedAndSolutionsReached: false,
    maxScansToExplodeReached: false,
    prunedSimilarIndexes: false,
    winningPlan: {
      isCached: false,
      stage: 'COLLSCAN',
      filter: { restaurant_id: { '$eq': '40361708' } },
      direction: 'forward'
    },
    rejectedPlans: []
  },
  executionStats: {
    executionSuccess: true,
    nReturned: 1,
    executionTimeMillis: 22,
    totalKeysExamined: 0,
    totalDocsExamined: 25359,
    executionStages: {
      isCached: false,
      stage: 'COLLSCAN',
      filter: { restaurant_id: { '$eq': '40361708' } },
      nReturned: 1,
      executionTimeMillisEstimate: 20,
      works: 25360,
      advanced: 1,
      needTime: 25358,
      needYield: 0,
      saveState: 1,
      restoreState: 1,
      isEOF: 1,
      direction: 'forward',
      docsExamined: 25359
    }
  },
  queryShapeHash: 'BB4853BE7BAEE5AC49F54556E5F106E34AE79411503664D5189653C291017885',
  command: {
    find: 'restaurants',
    filter: { restaurant_id: '40361708' },
    '$db': 'sample_restaurants'
  },
  serverInfo: {
    host: 'course',
    port: 27017,
    version: '8.0.26',
    gitVersion: 'a2c24805265db119cc39f2f54e067b4294a4ddd2'
  },
  serverParameters: {
    internalQueryFacetBufferSizeBytes: 104857600,
    internalQueryFacetMaxOutputDocSizeBytes: 104857600,
    internalLookupStageIntermediateDocumentMaxSizeBytes: 104857600,
    internalDocumentSourceGroupMaxMemoryBytes: 104857600,
    internalQueryMaxBlockingSortMemoryUsageBytes: 104857600,
    internalQueryProhibitBlockingMergeOnMongoS: 0,
    internalQueryMaxAddToSetBytes: 104857600,
    internalDocumentSourceSetWindowFieldsMaxMemoryBytes: 104857600,
    internalQueryFrameworkControl: 'trySbeRestricted',
    internalQueryPlannerIgnoreIndexWithCollationForRegex: 1
  },
  ok: 1
}
``` 
