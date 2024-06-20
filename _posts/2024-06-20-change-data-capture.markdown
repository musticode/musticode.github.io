---
layout: post
title:  "CDC Debezium integration with SpringBoot"
date:   2024-06-19 20:03:36 +0530
categories: SpringBoot Java Kafka
---

CDC Debezium integration with SpringBoot 

### CDC
Change Data Capture (CDC) is a technique and a design pattern. We often use it to replicate data between databases in real-time. <br>

We can also track data changes written to a source database and automatically sync target databases. CDC enables incremental loading and eliminates the need for bulk load updating.

### Debezium
Based on Apache Kafka, Debezium is an open-source platform for CDC. Its main purpose is to create a transaction log that contains all row-level modifications committed to each source database table. Based on incremental changes in the data, any application that is listening to these events can take the necessary steps.



### Project overview

docker-compose.yml
```yml
version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - 52181:2181

  kafka:
    image: confluentinc/cp-kafka:7.0.1
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_INTERNAL:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092,PLAINTEXT_INTERNAL://kafka:29092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1

  kafka-admin:
    image: provectuslabs/kafka-ui
    container_name: kafka-admin
    ports:
      - "9091:8080"
    restart: always
    depends_on:
      - kafka
      - zookeeper
    environment:
      - KAFKA_CLUSTERS_0_NAME=local
      - KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:29092
      - KAFKA_CLUSTERS_0_ZOOKEEPER=zookeeper:2181

  db:
    platform: linux/x86_64
    image: debezium/example-postgres
    restart: always
    environment:
      POSTGRES_PASSWORD: example
    ports:
      - 5432:5432
    extra_hosts:
      - "host.docker.internal:host-gateway"
    command:
      - "postgres"
      - "-c"
      - "wal_level=logical"
    volumes:
      - ./resource/init.sql:/docker-entrypoint-initdb.d/create-db-tables.sql

  elasticsearch:
    image: elasticsearch:8.8.0
    ports:
      - 9200:9200
      - 9300:9300
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"

  adminer:
    image: adminer
    restart: always
    ports:
      - 8001:8080

  kafka_connect:
    container_name: kafka_connect
    image: debezium/connect
    links:
      - db
      - kafka
    ports:
      - '8083:8083'
    environment:
      - BOOTSTRAP_SERVERS=kafka:29092
      - GROUP_ID=medium_debezium
      - CONFIG_STORAGE_TOPIC=my_connect_configs
      - OFFSET_STORAGE_TOPIC=my_connect_offsets
      - STATUS_STORAGE_TOPIC=my_connect_statuses
```

### Kafka UI

```
http://localhost:9091/ui/clusters/local/all-topics?perPage=25
```

### Database
- Set connection username : `postgres` and password : `example`
- Create a valid database `create database debezium_demo`
- Run query to create database
- properties.yml file example
```
url: jdbc:postgresql://localhost:5432/debezium_demo
username: postgres
password: example
```


### Connector Requests


**Connector information:**
- Connector should be created at first. It seems like you're working with the Debezium connector, which is used for change data capture (CDC) from database management systems like PostgreSQL. Debezium is commonly used in streaming data pipelines to capture and propagate database changes to other systems or applications in real-time.
  To configure a Debezium connector for PostgreSQL, you typically need to provide configuration parameters such as database connection details, the replication slot name, and other connector-specific settings. Below is an example configuration for setting up a Debezium connector for PostgreSQL:

```bash
{
    "name": "product-connector",
    "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "tasks.max": "1",
        "database.history.kafka.bootstrap.servers": "kafka:9092",
        "database.hostname": "db", 
        "database.port": "5432",
        "database.user": "postgres",
        "database.password": "example",
        "database.dbname": "debezium_demo",
        "database.server.name": "postgres",
        "tombstones.on.delete" : false,    
        "topic.prefix" : "product",
        "table.inclde.list" : "public.product",
        "heartbeat.interval.ms" : "5000",
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
        "key.converter.schemas.enable": false,
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter.schemas.enable": false,
        "plugin.name" : "pgoutput",
        "slot.name": "unique_slot_name"
        
    }
}
```

- `connector.class` : Specifies the class for the PostgreSQL connector.
- `tasks.max` : Defines the maximum number of tasks to create for this connector.
- `database.hostname`, `database.port`, `database.user`, `database.password`, `database.dbname` : Connection details for PostgreSQL.
- `database.server.name`: A unique identifier for the connector instance. Server's name can be found in properties for database
- `slot.name` : The replication slot name used by the connector. Make sure it's unique for each connector.
- `plugin.name` : Specifies the Debezium plugin to use for capturing changes.
- `schema.include`, table.include.list: Specify the schema and tables to include in the CDC.
- `snapshot.mode` : Defines how the initial snapshot of data should be taken.
- `heartbeat.interval.ms` : Interval for sending heartbeat messages to PostgreSQL.
- `heartbeat.action.query` : SQL query used for heartbeat messages.
- `snapshot.lock.timeout.ms` : Timeout for acquiring locks during the snapshot.
- `snapshot.select.statement.overrides` : Overrides the default SELECT statement used during the snapshot.
- `database.history.kafka.bootstrap.servers": "kafka:9092" ` : Kafka broker information
- `"key.converter": "org.apache.kafka.connect.json.JsonConverter"` and `"value.converter": "org.apache.kafka.connect.json.JsonConverter"` : Kafka serializing configs




#### CURL Requests:

GET Connector :
```bash
curl --request GET \
  --url http://localhost:8083/connectors/product-connector
```

POST - add a connector
```bash
localhost:8083/connectors
```

CURL Request :
```bash
curl --location 'localhost:8083/connectors/' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--header 'Cookie: csrftoken=M7jtiBvQrP5MVhm9kN1nbq8hNFgi0lRS' \
--data '{
    "name": "product-connector",
    "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "tasks.max": "1",
        "database.history.kafka.bootstrap.servers": "kafka:9092",
        "database.hostname": "db", 
        "database.port": "5432",
        "database.user": "postgres",
        "database.password": "example",
        "database.dbname": "debezium_demo",
        "database.server.name": "postgres",
        "tombstones.on.delete" : false,    
        "topic.prefix" : "product",
        "table.inclde.list" : "public.product",
        "heartbeat.interval.ms" : "5000",
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
        "key.converter.schemas.enable": false,
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter.schemas.enable": false,
        "plugin.name" : "pgoutput",
        "slot.name": "unique_slot_name"
        
    }
}'
```

DELETE Connector
```
curl --request DELETE \
  --url http://localhost:8083/connectors/product-connector
```


Extra information for `slot.name`:
- The error message indicates that the creation of a replication slot failed due to a slot with the name "debezium" already existing. This typically happens when attempting to set up multiple connectors for the same database host using the same replication slot name.
- To resolve this issue, you need to ensure that each connector you set up for the same database host uses a distinct replication slot name. This can be achieved by configuring a unique replication slot name for each connector.
- You can specify the replication slot name in the Debezium PostgreSQL connector configuration. Look for the slot.name parameter and ensure that it is set to a unique value for each connector.



**Before null problem**
<br>
`alter table product replica identity full` : run this command in database console


### Project Running Configuration
- Build project
- Open Docker in local machine
- Run docker-compose up -d for `docker-compose.yml` file, this can be changed for other configurations
- Run project
- Open `http://localhost:9091/ui/clusters/local/all-topics?perPage=25` to see topics and messages on Kafka
- Send CURL to create a connector, post connector
- Open messages in kafka ui localhost:9091, `product.public.product` topic should be seen
- Send request to create Product
```bash
curl --location 'localhost:8080/api/v1/product/add-product' \
--header 'Content-Type: application/json' \
--header 'Cookie: csrftoken=M7jtiBvQrP5MVhm9kN1nbq8hNFgi0lRS' \
--data '{
    "name" : "pant",
    "price" : 150,
    "stock" : 4
}'
```

- Check debezium topic [CREATE]
```json
{
	"before": null,
	"after": {
		"id": 25,
		"name": "pant",
		"price": "Opg=",
		"stock": 4
	},
	"source": {
		"version": "2.5.0.Final",
		"connector": "postgresql",
		"name": "product",
		"ts_ms": 1710684511415,
		"snapshot": "false",
		"db": "debezium_demo",
		"sequence": "[\"25659272\",\"25659664\"]",
		"schema": "public",
		"table": "product",
		"txId": 771,
		"lsn": 25659664,
		"xmin": null
	},
	"op": "c",
	"ts_ms": 1710684511616,
	"transaction": null
}
```

- Make an update operation to product:
```bash
curl --location --request PUT 'localhost:8080/api/v1/product/update-product/14' \
--header 'Content-Type: application/json' \
--header 'Cookie: csrftoken=M7jtiBvQrP5MVhm9kN1nbq8hNFgi0lRS' \
--data '{
    "name" : "pant",
    "price" : 1512,
    "stock" : 6
}'
```

- Check messages in debezium topic: `product.public.product`
```json
{
	"before": {
		"id": 14,
		"name": "pant",
		"price": "Opg=",
		"stock": 4
	},
	"after": {
		"id": 14,
		"name": "pant",
		"price": "Ak6g",
		"stock": 6
	},
	"source": {
		"version": "2.5.0.Final",
		"connector": "postgresql",
		"name": "product",
		"ts_ms": 1710680240132,
		"snapshot": "false",
		"db": "debezium_demo",
		"sequence": "[\"25645448\",\"25645504\"]",
		"schema": "public",
		"table": "product",
		"txId": 760,
		"lsn": 25645504,
		"xmin": null
	},
	"op": "u",
	"ts_ms": 1710680240470,
	"transaction": null
}
```