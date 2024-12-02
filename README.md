# Apache Paimon Playground with Flink & Trino

![image](https://github.com/user-attachments/assets/52f8846b-a1be-4b71-af44-12ca307b696c)

## How to use

> NOTE: Since some of the links to the official Paimon files are not working, I've put the files into this repo. However, some of the files are huge, so I put them in via LFS, so be sure to install `git-lfs`.

Trino driver is in `paimon-trino-427-0.8-20241112.000605-197-plugin.tar.gz`.

```bash
tar -zxvf paimon-trino-427-0.8-20241112.000605-197-plugin.tar.gz
```

Then it will run normally with `docker compose up -d`.

```bash
docker compose up -d
```

### Flink

Let's start by connecting to Flink SQL.

```bash
docker compose exec flink-jobmanager ./bin/sql-client.sh
./bin/sql-client.sh
```

To write data using Flink we first need to create the correct catalog.

```sql
CREATE CATALOG my_catalog WITH (
    'type' = 'paimon',
    'warehouse' = 's3://warehouse/flink',
    's3.endpoint' = 'http://storage:9000',
    's3.access-key' = 'admin',
    's3.secret-key' = 'password',
    's3.path.style.access' = 'true'
);
```

As shown in the above commands, we're using the `MinIO` as an `S3` to store the Paimon.

The next step in creating the table and writing the data is quite simple, just run the commands according to the [official documentat](https://paimon.apache.org/docs/0.9/flink/quick-start/).

```sql
USE CATALOG my_catalog;

-- create a word count table
CREATE TABLE word_count (
    word STRING PRIMARY KEY NOT ENFORCED,
    cnt BIGINT
);

-- create a word data generator table
CREATE TEMPORARY TABLE word_table (
    word STRING
) WITH (
    'connector' = 'datagen',
    'fields.word.length' = '1'
);

-- paimon requires checkpoint interval in streaming mode
SET 'execution.checkpointing.interval' = '10 s';

-- write streaming data to dynamic table
INSERT INTO word_count SELECT word, COUNT(*) FROM word_table GROUP BY word;
```

Then we actually read it and see the result of what we've written.

```sql
-- use tableau result mode
SET 'sql-client.execution.result-mode' = 'tableau';

-- switch to batch mode
RESET 'execution.checkpointing.interval';
SET 'execution.runtime-mode' = 'batch';

-- olap query the table
SELECT * FROM word_count;
```

### Trino

Let's go to Trino's cli first.

```bash
docker compose exec trino trino
```

Trino's paimon catalog is already set up, but I didn't add a new schema but just used the default one.

So we can query the Flink write result directly.

```sql
SELECT * FROM paimon.default.word_count;
```

We should see something similar to the Flink query.

### StarRocks

This is an extra, just to show how much attention Paimon is getting now that many New SQL databases are starting to support it.

Prepare a `mysql` client locally to connect to StarRocks.

```bash
mysql -P 9030 -h 127.0.0.1 -u root --prompt="StarRocks > "
```

We still need to create a catalog.

```sql
CREATE EXTERNAL CATALOG paimon_catalog_flink
PROPERTIES
(
    "type" = "paimon",
    "paimon.catalog.type" = "filesystem",
    "paimon.catalog.warehouse" = "s3://warehouse/flink",
    "aws.s3.enable_path_style_access" = "true",
    "aws.s3.endpoint" = "http://storage:9000",
    "aws.s3.access_key" = "admin",
    "aws.s3.secret_key" = "password"
);
```

The mysql client should not support Trino's table locator format: `<catalog>. <schema>. <table>`, so we have to switch to the db before we can query.

```sql
USE paimon_catalog_flink.default;
SELECT * FROM word_count;
```

The results here will be similar to the above.

## MySQL Catalog

Flink supports JDBC metastore, which allows using MySQL to persist Paimon metadata seamlessly. Here's an example of creating a MySQL-based catalog.

```sql
CREATE CATALOG mysql_catalog WITH (
    'type' = 'paimon',
    'metastore' = 'jdbc',
    'uri' = 'jdbc:mysql://mysql:3306/catalog',
    'jdbc.user' = 'root',
    'jdbc.password' = 'example',
    'warehouse' = 's3://warehouse/mysql',
    's3.endpoint' = 'http://storage:9000',
    's3.access-key' = 'admin',
    's3.secret-key' = 'password',
    's3.path.style.access' = 'true'
);
```

However, **Trino does not support JDBC metastore** for Paimon so far.
