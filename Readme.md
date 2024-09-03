# install docker and docker-compose

`sudo yum install docker -y`

`sudo service docker start`

##### make docker autostart

`sudo chkconfig docker on`

##### docker-compose (latest version)

`sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose`

##### Fix permissions after download

`sudo chmod +x /usr/local/bin/docker-compose`

##### Verify

`docker-compose version`

# Create RDS cluster

# Enable CDC

`rds.logical_replication=1`

# Create MSK Cluster.

# Enable Auto create to true

if not you need to create topics manually

##### cluster configureations

```
auto.create.topics.enable=true
default.replication.factor=2
min.insync.replicas=1
num.io.threads=8
num.network.threads=5
num.partitions=1
num.replica.fetchers=1
replica.lag.time.max.ms=30000
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
socket.send.buffer.bytes=102400
unclean.leader.election.enable=true
zookeeper.session.timeout.ms=18000
max.incremental.fetch.session.cache.slots=10000
delete.topic.enable=true
```

# Create Docker file to include Avro to debezium

use `Dockerfile` in this project

# Create `docker-compose` file

use `cdc-compose-with-schemaReg.yml` file in this project.
if you do not need AVRO or schema registry you can use `cdc-compose.yml`

# Run bellow URLs

#### Kafdrop

`http://<your ip>:9000`

#### Kafka-connect-UI

`http://<your ip>:8000`

#### schema registry

`http://<your ip>:8081/subjects`

Then

`subjects/cosmosdb.public.employees-value/versions/latest`

# create table

```
CREATE TABLE IF NOT EXISTS public.employees
(
    id smallint NOT NULL GENERATED ALWAYS AS IDENTITY ( INCREMENT 1 START 1 MINVALUE 1 MAXVALUE 32767 CACHE 1 ),
    name character varying COLLATE pg_catalog."default",
    city character varying COLLATE pg_catalog."default",
    CONSTRAINT employees_pkey PRIMARY KEY (id)
)

TABLESPACE pg_default;

ALTER TABLE IF EXISTS public.employees
    OWNER to postgres;
```

# create Connector

### with schema registry

```
name=cosmos-pg-connector
connector.class=io.debezium.connector.postgresql.PostgresConnector
database.dbname=cosmos
database.user=postgres
slot.name=cosmos
database.history.kafka.bootstrap.servers=PLAINTEXT://b-1.cdcclustermsk.7vyu01.c2.kafka.ap-southeast-1.amazonaws.com:9092,PLAINTEXT://b-2.cdcclustermsk.7vyu01.c2.kafka.ap-southeast-1.amazonaws.com:9092
database.history.kafka.topic=dbhistory.employees
database.server.name=pg-cosmos
database.hostname=cosmos-pg.cluster-c2qj6mbvxyfb.ap-southeast-1.rds.amazonaws.com
database.port=5481
database.password=passwprd##%%
poll.interval.ms=3000
table.include.list=public.employees
plugin.name=pgoutput
topic.prefix=cosmosdb
decimal.handling.mode=double
# for schema
key.converter.schemas.enable=true
key.converter=io.confluent.connect.avro.AvroConverter
key.converter.schema.registry.url=http://schema-registry:8081
value.converter.schemas.enable=true
value.converter.schema.registry.url=http://schema-registry:8081
value.converter=io.confluent.connect.avro.AvroConverter
snapshot.mode=initial
```

### without schema registry

```
name=cosmos-pg-connector
connector.class=io.debezium.connector.postgresql.PostgresConnector
database.dbname=cosmos
database.user=postgres
slot.name=cosmos
database.history.kafka.bootstrap.servers=PLAINTEXT://b-1.cdcclustermsk.7vyu01.c2.kafka.ap-southeast-1.amazonaws.com:9092,PLAINTEXT://b-2.cdcclustermsk.7vyu01.c2.kafka.ap-southeast-1.amazonaws.com:9092
database.history.kafka.topic=dbhistory.employees
database.server.name=pg-cosmos
database.hostname=cosmos-pg.cluster-c2qj6mbvxyfb.ap-southeast-1.rds.amazonaws.com
database.port=5481
database.password=1qazxsw2##%%
poll.interval.ms=3000
table.include.list=public.employees
plugin.name=pgoutput
topic.prefix=cosmosdb
decimal.handling.mode=double
snapshot.mode=initial
```

## To enable Before on update statments.

`ALTER TABLE public.employees REPLICA IDENTITY FULL;`

## sync connector

```
name=cosmos-mysql-sink-connector
connector.class=io.debezium.connector.jdbc.JdbcSinkConnector
tasks.max=1
topics=cosmosdb.public.employees

# Connection to MySQL (RDS)
connection.url=jdbc:mysql://cosmos-mysql.c2qj6mbvxyfb.ap-southeast-1.rds.amazonaws.com:3381/employeesdb
connection.username=admin
connection.password=1qazxsw2##%%

# Table settings
table.name.format=employees
delete.enabled=true
schema.evolution=basic
primary.key.mode=record_key
primary.key.fields=id
insert.mode=upsert

# Converter settings
key.converter=io.confluent.connect.avro.AvroConverter
key.converter.schema.registry.url=http://schema-registry:8081
value.converter.schema.registry.url=http://schema-registry:8081
value.converter=io.confluent.connect.avro.AvroConverter
```

## Health Check

`curl -s -X GET http://<your ip>:8083/connectors/cosmos-pg-connector/status`

#### Restart

`curl -X POST http://<your ip>:8083/connectors/cosmos-pg-connector/restart`

### Security group (for referance only)

Update IPs as required

![alt text](security-group.jpg 'security group')
