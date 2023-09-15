# Kafka Connect Environment for Nessie/Iceberg

### Step 1 - Create Iceberg Table

- Open up three terminals to the directory with the docker-compose.yaml and startup Nessie, Dremio and Minio

```bash
# In terminal #1
docker-compose up dremio
# In terminal #2
docker-compose up minio
# In terminal #3
docker-compose up nessie
```

- Once all started up open up minio at localhost:9000 login with admin/password and create a bucket called "warehouse"

- open up dremio and connect nessie source with the following
 - general settings
    - name: nessie
    - url: http://nessie:19120/api/v2
    - auth: non
 - storage settings
    - auth type: aws credentials
    - access key: admin
    - secret key: password
    - Root Path: /warehouse
    - connection properties:
        - fs.s3a.path.style.access=`true`
        - fs.s3a.endpoint=`minio:9000`
        - dremio.s3.compat=`true`
    - encrypt connection: unchecked

- create folder in your catalog called `db`

- run the following SQL 

```
CREATE TABLE default.events_list (
    id VARCHAR,
    type VARCHAR,
    ts TIMESTAMP,
    payload VARCHAR)
PARTITIONED BY (hours(ts));
```

## Step 2 - Spin Up Kafka and Kafka Connect

- open three more terminals

```
# In terminal #4
docker-compose up zookeeper
# In terminal #5
docker-compose up kafka
# In terminal #6
docker-compose up kafka-connect
```

- Create a kafka topic with the following command in a 7th terminal

```
docker-compose exec kafka kafka-topics.sh --create --bootstrap-server kafka:9092 --replication-factor 1 --partitions 1 --topic my_topic
```


## Step 3 - Kafka-Connect Configuration

Using curl or any http client make the follow REST call:

```
curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d '{
"name": "events-sink",
"config": {
    "connector.class": "io.tabular.iceberg.connect.IcebergSinkConnector",
    "tasks.max": "2",
    "topics": "my_topic",
    "iceberg.tables": "db.table",
    "iceberg.catalog.type": "nessie",
    "iceberg.catalog.uri": "http://nessie:19120",
    "iceberg.catalog.authentication.type": "NONE",
    "iceberg.catalog.warehouse": "s3a://warehouse",
    "iceberg.catalog.catalog-imp": "org.apache.iceberg.nessie.NessieCatalog",
    "iceberg.catalog.io-impl": "org.apache.iceberg.aws.s3.S3FileIO",
    "iceberg.catalog.client.region":"us-east-1",
    "iceberg.catalog.s3.endpoint": "http://minio:9000",
    "iceberg.catalog.s3.access-key-id":"admin",
    "iceberg.catalog.s3.secret-access-key":"password"
    ...
    }
}'
```

## Step 4 - Push data to topic

```
echo '{"schema": {"type": "string"}, "payload": "{\"id\": 1, \"name\": \"Alex Merced\"}"}' | docker-compose exec -T kafka kafka-console-producer --bootstrap-server kafka:9092 --topic my_topic
```

## Step 5 - Query table to see if data landed

Head back to dremio and query the table.