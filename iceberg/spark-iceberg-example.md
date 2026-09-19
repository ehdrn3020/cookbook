# Spark Iceberg 예제

> 작성일: 2026-04-07

## Spark Execute

- [Spark and Iceberg Quickstart - Apache Iceberg™](https://iceberg.apache.org/spark-quickstart/#creating-a-table)
- [Configuration - Apache Iceberg™](https://iceberg.apache.org/docs/latest/spark-configuration/#catalogs)

```javascript
./spark-shell \
  --packages org.apache.iceberg:iceberg-spark-runtime-3.4_2.12:1.5.0 \
  --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions \
  --conf spark.sql.catalog.my_lake=org.apache.iceberg.spark.SparkCatalog \
  --conf spark.sql.catalog.my_lake.type=hadoop \
  --conf spark.sql.catalog.my_lake.warehouse=hdfs://1.2.3.4:8020/user/iceberg/warehouse
  
packages org.apache.iceberg : 필수, spark_scala:iceberg 버전
conf spark.sql.extensions : Iceberg SQL 확장 기능을 Spark SQL에 붙이는 설정
conf spark.sql.catalog.my_lake : my_lake라는 이름의 catalog를 Iceberg catalog로 등록하는 설정
conf spark.sql.catalog.my_lake.type : catalog 종류를 지정하는 설정
conf spark.sql.catalog.my_lake.warehouse : catalog의 최상위 warehouse 경로 설정
```

## Spark Code

```javascript
sql("""
  CREATE TABLE my_lake.db.members (
    id bigint,
    name string,
    age int
  ) USING iceberg
""")

sql("""
  INSERT INTO my_lake.db.members
  VALUES (1, 'UserA', 20), (2, 'UserB', 30)
""")

val updates = Seq(
  (1L, "UserA", 21),
  (3L, "UserC", 25)
).toDF("id", "name", "age")
updates.createOrReplaceTempView("source_changes")

// MERGE INTO - id가 같으면 UPDATE, 없으면 INSERT
sql("""
  MERGE INTO my_lake.db.members AS target
  USING source_changes AS source
  ON target.id = source.id
  WHEN MATCHED THEN
    UPDATE SET
      target.name = source.name,
      target.age = source.age
  WHEN NOT MATCHED THEN
    INSERT (id, name, age)
    VALUES (source.id, source.name, source.age)
""")

// Schema Evolution
sql("""
  ALTER TABLE my_lake.db.members
  ADD COLUMNS (email string)
""")
```
