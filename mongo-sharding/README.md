# MongoDB sharding

## Запуск контейнеров

```shell
docker compose up -d
```

## Инициализация config server

```shell
docker compose exec -T configsvr mongosh --port 27017 --quiet <<EOF
rs.initiate({
  _id: "config_server",
  configsvr: true,
  members: [
    { _id: 0, host: "configsvr:27017" }
  ]
})
EOF
```

После инициализации config server подождите несколько секунд, чтобы `mongos` подключился к нему.

## Инициализация shard1

```shell
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
rs.initiate({
  _id: "shard1",
  members: [
    { _id: 0, host: "shard1:27018" }
  ]
})
EOF
```

## Инициализация shard2

```shell
docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF
rs.initiate({
  _id: "shard2",
  members: [
    { _id: 0, host: "shard2:27019" }
  ]
})
EOF
```

После инициализации shard-ов подождите несколько секунд, чтобы single-member replica set-ы стали primary.

## Добавление шардов

Команды можно выполнить через любой роутер. Ниже используется `mongos-router-1`.

```shell
docker compose exec -T mongos-router-1 mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/shard1:27018")
sh.addShard("shard2/shard2:27019")
EOF
```

## Создание БД и коллекции

```shell
docker compose exec -T mongos-router-1 mongosh --port 27020 --quiet <<EOF
use somedb
db.createCollection("helloDoc")
db.helloDoc.createIndex({ age: "hashed" })
sh.enableSharding("somedb")
sh.shardCollection("somedb.helloDoc", { age: "hashed" })
EOF
```

## Заполнение тестовыми данными

```shell
docker compose exec -T mongos-router-1 mongosh --port 27020 --quiet <<EOF
use somedb
for (let i = 0; i < 1000; i++) {
  db.helloDoc.insertOne({ age: i, name: "ly" + i })
}
EOF
```

## Проверка

Проверить статус шардирования:

```shell
docker compose exec -T mongos-router-1 mongosh --port 27020 --quiet <<EOF
sh.status()
EOF
```

Проверить количество документов в `somedb.helloDoc` через роутер:

```shell
docker compose exec -T mongos-router-1 mongosh --port 27020 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

Проверить количество документов на первом шарде напрямую:

```shell
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

Проверить количество документов на втором шарде напрямую:

```shell
docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

API доступен на `http://localhost:8080`, Swagger - на `http://localhost:8080/docs`.

Главная страница API (`http://localhost:8080`) показывает общее количество документов в `somedb.helloDoc`
