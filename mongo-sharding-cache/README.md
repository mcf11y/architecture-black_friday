# MongoDB sharding, replica sets and Redis cache

Проект реализует схему из задания 4: шардирование с репликацией из второго варианта схемы и Redis-кеш для эндпоинта `/<collection_name>/users`.

- `configsvr` - config server.
- `shard1` - replica set из `s1-r1`, `s1-r2`, `s1-r3`.
- `shard2` - replica set из `s2-r1`, `s2-r2`, `s2-r3`.
- `mongos-router-1`, `mongos-router-2` - роутеры.
- `redis` - кеш.
- `pymongo_api` - приложение, подключенное к `mongos` и Redis.

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

```shell
docker compose exec -T mongos-router-1 mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/s1-r1:27018,s1-r2:27018,s1-r3:27018")
sh.addShard("shard2/s2-r1:27019,s2-r2:27019,s2-r3:27019")
EOF
```

## Создание БД и коллекции

База данных называется `somedb`, коллекция - `helloDoc`.

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

## Проверка MongoDB

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

Главная страница API (`http://localhost:8080`) показывает общее количество документов в `somedb.helloDoc`, количество документов на каждом shard-е, количество реплик и `cache_enabled: true`.

## Проверка кеширования

Кеширование включено переменной окружения:

```yaml
REDIS_URL: "redis://redis:6379"
```

Эндпоинт с кешем: `GET /<collection_name>/users`. Для тестовой коллекции:

```shell
curl -w "\ntime_total=%{time_total}\n" -o /dev/null -s http://localhost:8080/helloDoc/users
curl -w "\ntime_total=%{time_total}\n" -o /dev/null -s http://localhost:8080/helloDoc/users
```

Первый запрос выполняется примерно за 1 секунду из-за искусственной задержки в приложении. Второй и последующие запросы должны выполняться быстрее `100ms`, пока кеш не истек.
