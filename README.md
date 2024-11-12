# Paimon Playground

- [paimon-trino-427-0.8-20241112.000605-197-plugin.tar.gz](https://repository.apache.org/content/groups/snapshots/org/apache/paimon/paimon-trino-427/0.8-SNAPSHOT/paimon-trino-427-0.8-20241112.000605-197-plugin.tar.gz)
    - `tar -zxvf paimon-trino-427-0.8-20241112.000605-197-plugin.tar.gz`

## Flink

https://paimon.apache.org/docs/0.9/flink/quick-start/

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

## Trino

```sql
SELECT * FROM paimon.default.word_count;
```

The result should be as follows.

```
 word |  cnt
------+--------
 0    | 311054
 1    | 312766
 2    | 312566
 3    | 311228
 4    | 311448
 5    | 311613
 6    | 311479
 7    | 312043
 8    | 312506
 9    | 311642
 a    | 310712
 b    | 312189
 c    | 312293
 d    | 312391
 e    | 312015
 f    | 312055
(16 rows)
```

## StarRocks

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

Switch to db.

```sql
use paimon_catalog_flink.default;
```

Query the table.

```sql
select * from word_count;
```


## Issue List

> Path missing in file system location: `s3://warehouse`

warehouse=s3://<namespace>/<warehouse dir>

[Reference link](https://github.com/apache/paimon-trino/issues/26#issuecomment-2187113570)



> Query 20241112_023941_00000_bdn6f failed: org.apache.paimon.fs.UnsupportedSchemeException: Could not find a file io implementation for scheme 's3' in the classpath. TrinoFileIOLoader also cannot access this path.  Hadoop FileSystem also cannot access this path 's3://minio/warehouse'.
