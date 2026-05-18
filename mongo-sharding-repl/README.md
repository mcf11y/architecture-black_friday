# MongoDB sharding with replica sets

## Запуск

Если стенд уже запускался и нужно начать с чистого состояния:

```shell
docker compose down -v
```

Запустите контейнеры:

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

## Инициализация shard1

```shell
docker compose exec -T s1-r1 mongosh --port 27018 --quiet <<EOF
rs.initiate({
  _id: "shard1",
  members: [
    { _id: 0, host: "s1-r1:27018" },
    { _id: 1, host: "s1-r2:27018" },
    { _id: 2, host: "s1-r3:27018" }
  ]
})
EOF
```

## Инициализация shard2

```shell
docker compose exec -T s2-r1 mongosh --port 27019 --quiet <<EOF
rs.initiate({
  _id: "shard2",
  members: [
    { _id: 0, host: "s2-r1:27019" },
    { _id: 1, host: "s2-r2:27019" },
    { _id: 2, host: "s2-r3:27019" }
  ]
})
EOF
```

Подождите несколько секунд, пока replica set-ы выберут primary.

## Добавление шардов

Команды выполняются через любой `mongos`. Ниже используется `mongos-router-1`.

```shell
docker compose exec -T mongos-router-1 mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/s1-r1:27018,s1-r2:27018,s1-r3:27018")
sh.addShard("shard2/s2-r1:27019,s2-r2:27019,s2-r3:27019")
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

## Заполнение данными

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

Проверить количество документов через роутер:

```shell
docker compose exec -T mongos-router-1 mongosh --port 27020 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

Проверить количество документов на `shard1`:

```shell
docker compose exec -T s1-r1 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

Проверить количество документов на `shard2`:

```shell
docker compose exec -T s2-r1 mongosh --port 27019 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

Проверить количество реплик:

```shell
docker compose exec -T s1-r1 mongosh --port 27018 --quiet <<EOF
rs.status().members.length
EOF

docker compose exec -T s2-r1 mongosh --port 27019 --quiet <<EOF
rs.status().members.length
EOF
```
API доступен на `http://localhost:8080`, Swagger - на `http://localhost:8080/docs`.

Главная страница API (`http://localhost:8080`) показывает общее количество документов в `somedb.helloDoc`