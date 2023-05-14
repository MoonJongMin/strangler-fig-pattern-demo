# Strangler Fig Pattern Demo

## Build applications

Before being able to spin up the docker-compose based demo environment please make sure to successfully **build all 3 projects - the petclinic monolith based on Spring, the Apache Kafka Streams application and the owner microservice powered by Quarkus:**

```
./build-applications.sh
```

## Run with Docker

Spin up the demo environment by means of Docker compose:

```
docker-compose up
```

## Strangler Fig Proxy

This demo uses nginx and depending on the request, routes either towards petclinic monolith or the owner microservice.

1. The proxy serves a static index page at `http://localhost` just to verify it's up and running.

2. The initial proxy configuration - see `docker/.config/nginx/nginx_initial.conf` - routes all requests starting with `http://localhost/petclinic` to the monolithic application.

3. As a first adaption, the proxy is reconfigured - see `docker/.config/nginx/nginx_read.conf` - to route a specific read request i.e. the owner search from the monolith to the corresponding microservice.

_NOTE: In order for this to work the CDC setup (from monolith -> microservice) needs to be configured properly based on Apache Kafka Connect (see below)_

4. As a second adaption, the proxy is reconfigured - see `docker/.config/nginx/nginx_initial.conf` - to route owner edit requests to the owner microservice as well.

_NOTE: In order for this to work the CDC setup (from microservice -> monolith) needs to be configured properly based on Apache Kafka Connect (see below)_

## Proxy Reconfiguration

With the docker compose stack up and running, the proxy can be reconfigured by running the following simple script with one of the 3 support parameters:

```
./proxy_config.sh initial | read | read_write
```

## Apache Kafka Connect Setup for CDC Pipelines

### CDC from monolith (MySQL) -> microservice (MongoDB)

Below command needs httpie tool on your laptop. If you don't have, install httpie.

1. Create MySQL source connector for owners + pets tables:

```
http POST http://localhost:8083/connectors/ < register-mysql-source-owners-pets.json
```
When you run the above command, you would get the below response. If you don't, restart connect container.
This command will debezium run CDC from mysql then put the message into kafka topic.
```
HTTP/1.1 201 Created
Content-Length: 780
Content-Type: application/json
Date: Sun, 14 May 2023 04:46:27 GMT
Location: http://localhost:8083/connectors/petclinic-owners-pets-mysql-src-001
Server: Jetty(9.4.33.v20201020)

{
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "database.history.kafka.bootstrap.servers": "kafka:9092",
        "database.history.kafka.topic": "schema-changes.petclinic",
        "database.hostname": "mysql",
        "database.include": "petclinic",
        "database.password": "debezium",
        "database.port": "3306",
        "database.server.id": "12345",
        "database.server.name": "mysql1",
        "database.user": "root",
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
        "key.converter.schemas.enable": "false",
        "name": "petclinic-owners-pets-mysql-src-001",
        "table.include.list": "petclinic.owners,petclinic.pets",
        "tasks.max": "1",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter.schemas.enable": "false"
    },
    "name": "petclinic-owners-pets-mysql-src-001",
    "tasks": [],
    "type": "source"
}
```

2. Create MongoDB sink connector for pre-joined owner with pets aggregates:

```
http POST http://localhost:8083/connectors/ < register-mongodb-sink-owners-pets.json
```
You would get the below response. This command will get message from kafka topic then put the message into MongoDB.
```
HTTP/1.1 201 Created
Content-Length: 1346
Content-Type: application/json
Date: Sun, 14 May 2023 04:46:54 GMT
Location: http://localhost:8083/connectors/petclinic-owners-pets-mongodb-sink-001
Server: Jetty(9.4.33.v20201020)

{
    "config": {
        "connection.uri": "mongodb://mongodb:27017",
        "connector.class": "com.mongodb.kafka.connect.MongoSinkConnector",
        "database": "petclinic",
        "document.id.strategy": "com.mongodb.kafka.connect.sink.processor.id.strategy.ProvidedInKeyStrategy",
        "field.renamer.mapping": "[{\"oldName\":\"value.owner.id\", \"newName\":\"owner_id\"}]",
        "key.converter": "org.apache.kafka.connect.storage.StringConverter",
        "name": "petclinic-owners-pets-mongodb-sink-001",
        "post.processor.chain": "com.mongodb.kafka.connect.sink.processor.BlockListValueProjector,com.mongodb.kafka.connect.sink.processor.field.renaming.RenameByMapping",
        "tasks.max": "1",
        "topics": "kstreams.owners-with-pets",
        "transforms": "createkey,flatkey,renameid",
        "transforms.createkey.fields": "owner",
        "transforms.createkey.type": "org.apache.kafka.connect.transforms.ValueToKey",
        "transforms.flatkey.delimiter": "_",
        "transforms.flatkey.type": "org.apache.kafka.connect.transforms.Flatten$Key",
        "transforms.renameid.renames": "owner_id:_id",
        "transforms.renameid.type": "org.apache.kafka.connect.transforms.ReplaceField$Key",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter.schemas.enable": "false",
        "value.projection.list": "pets.id,pets.owner_id,pets.birth_date,pets.type_id",
        "value.projection.type": "BlockList"
    },
    "name": "petclinic-owners-pets-mongodb-sink-001",
    "tasks": [],
    "type": "sink"
}
```
## Apache Kafka Connect Setup for CDC Pipelines

### CDC from microservice (MongoDB) -> monolith (MySQL)

1. Before processing writes for owner data at the microservice in MongoDB update the MySQL source connector from the monolith database to ignore changes happening in the owners table. Otherwise we would try to do CDC for the same data from both sides which would lead to a propagation cycle.

```
http PUT http://localhost:8083/connectors/petclinic-owners-pets-mysql-src-001/config < update-mysql-source-owners-pets.json
```

2. Create MongoDB source connector  to capture data changes in the microservice

```
http POST http://localhost:8083/connectors/ < register-mongodb-source-owners.json
```

3. Create MySQL JDBC sink connector  to propagate changes from the microservice into the monolith's database

```
http POST http://localhost:8083/connectors/ < register-jdbc-mysql-sink-owners.json
```

## Consume messages from CDC-related Apache Kafka topics
This command is to get data from kafka topics and not tightly related to thid demo.
When you run the below commands, if you have an error response that "network XXX not found.", run the command "docker network ls" and than find right network name for this demo.

Listing kafka meta information to find topics in a kafka broker.
```
docker run --tty --rm \
    --network strangler-fig-pattern-demo_default \
    debezium/tooling:1.1 \
    kafkacat -b kafka:9092 -L
```

Getting messages from each kafka topic.
```
docker run --tty --rm \
    --network strangler-fig-pattern-demo_default \
    debezium/tooling:1.1 \
    kafkacat -b kafka:9092 -C -t mysql1.petclinic.owners -o beginning -q | jq .
```

```
docker run --tty --rm \
    --network strangler-fig-pattern-demo_default \
    debezium/tooling:1.1 \
    kafkacat -b kafka:9092 -C -t mysql1.petclinic.pets -o beginning -q | jq .
```

```
docker run --tty --rm \
    --network strangler-fig-pattern-demo_default \
    debezium/tooling:1.1 \
    kafkacat -b kafka:9092 -C -t kstreams.owners-with-pets -o beginning -q | jq .
 ```

 ```
docker run --tty --rm \
    --network strangler-fig-pattern-demo_default \
    debezium/tooling:1.1 \
    kafkacat -b kafka:9092 -C -t mongodb.petclinic.kstreams.owners-with-pets -o beginning -q | jq .
```
