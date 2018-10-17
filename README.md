## Clone ksql repository

```
git clone https://github.com/confluentinc/ksql.git ksql && cd $_

git checkout 5.0.0-post
```

## Start docker-compose

```
cd docs/tutorials/

docker-compose up -d
```

### Generate pageviews data

```
docker run --network tutorials_default --rm --name datagen-pageviews \
    confluentinc/ksql-examples:5.0.0 \
    ksql-datagen \
        bootstrap-server=kafka:39092 \
        quickstart=pageviews \
        format=delimited \
        topic=pageviews \
        maxInterval=500
```

### Generate users data

```
docker run --network tutorials_default --rm --name datagen-users \
    confluentinc/ksql-examples:5.0.0 \
    ksql-datagen \
        bootstrap-server=kafka:39092 \
        quickstart=users \
        format=json \
        topic=users \
        maxInterval=100
```

## Start KSQL CLI

```
docker run --network tutorials_default --rm --interactive --tty \
     confluentinc/cp-ksql-cli:5.0.0 \
     http://ksql-server:8088
```

- `CREATE STREAM pageviews_original (viewtime bigint, userid varchar, pageid varchar) WITH \
(kafka_topic='pageviews', value_format='DELIMITED');`
- `CREATE TABLE users_original (registertime BIGINT, gender VARCHAR, regionid VARCHAR, userid VARCHAR) WITH \
(kafka_topic='users', value_format='JSON', key = 'userid');`

- `SHOW STREAMS;`
- `SHOW TABLES;`

- `SELECT pageid FROM pageviews_original LIMIT 3;`

#### Create enriched streams

```sql
CREATE STREAM pageviews_enriched AS \
      SELECT users_original.userid AS userid, pageid, regionid, gender \
      FROM pageviews_original \
      LEFT JOIN users_original \
        ON pageviews_original.userid = users_original.userid;
```

```sql
CREATE STREAM pageviews_female AS \
      SELECT * FROM pageviews_enriched \
      WHERE gender = 'FEMALE';
```

```sql
CREATE STREAM pageviews_female_like_89 \
        WITH (kafka_topic='pageviews_enriched_r8_r9') AS \
      SELECT * FROM pageviews_female \
      WHERE regionid LIKE '%_8' OR regionid LIKE '%_9';
```

```sql
CREATE TABLE pageviews_regions \
        WITH (VALUE_FORMAT='avro') AS \
      SELECT gender, regionid , COUNT(*) AS numusers \
      FROM pageviews_enriched \
        WINDOW TUMBLING (size 30 second) \
      GROUP BY gender, regionid \
      HAVING COUNT(*) > 1;
```

- `SELECT gender, regionid, numusers FROM pageviews_regions LIMIT 5;`

### Insert into streams

```
docker run --network tutorials_default --rm  --name datagen-orders-local \
    confluentinc/ksql-examples:5.0.0 \
    ksql-datagen \
        quickstart=orders \
        format=avro \
        topic=orders_local \
        bootstrap-server=kafka:39092 \
        schemaRegistryUrl=http://schema-registry:8081
```

```
docker run --network tutorials_default --rm --name datagen-orders_3rdparty \
    confluentinc/ksql-examples:5.0.0 \
    ksql-datagen \
        quickstart=orders \
        format=avro \
        topic=orders_3rdparty \
        bootstrap-server=kafka:39092 \
        schemaRegistryUrl=http://schema-registry:8081
```

### Create streams

```sql
CREATE STREAM ORDERS_SRC_LOCAL \
        WITH (KAFKA_TOPIC='orders_local', VALUE_FORMAT='AVRO');

CREATE STREAM ORDERS_SRC_3RDPARTY \
        WITH (KAFKA_TOPIC='orders_3rdparty', VALUE_FORMAT='AVRO');
```

### Create 'All orders' stream

```sql
CREATE STREAM ALL_ORDERS AS SELECT 'LOCAL' AS SRC, * FROM ORDERS_SRC_LOCAL;
```

```sql
INSERT INTO ALL_ORDERS SELECT '3RD PARTY' AS SRC, * FROM ORDERS_SRC_3RDPARTY;
```

*Reference*: https://docs.confluent.io/current/ksql/docs/tutorials/basics-docker.html#ksql-quickstart-docker
