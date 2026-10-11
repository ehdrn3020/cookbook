# Maintenance

> 작성일: 2026-04-14

- Iceberg는 데이터를 append하고 snapshot을 계속 쌓는 구조라서, 오래된 snapshot / metadata / 안 쓰는 파일을 주기적으로 청소
- append-only + snapshot 기반
- Expire Snapshots / Remove old metadata files / Delete orphan files 주기적 실행 필요
- Expire Snapshots : Java API로 실행 가능 / Spark SQL 프로시저로도 가능

## 세 작업의 성격이 다르다

| 작업 | 실행 방식 | 주기 |
| --- | --- | --- |
| Expire Snapshots | `CALL` 프로시저 or Java API | 매일 |
| Remove old metadata files | **테이블 속성** — 켜두면 커밋마다 자동 | 1회 설정 |
| Delete orphan files | `CALL` 프로시저 (Spark Actions 필요) | 주 1회 |

- 2번만 실행하는 명령이 아니라 속성이다. 나머지 둘은 보통 Spark 배치 잡 하나로 묶어 Airflow/cron으로 돌린다.
- `CALL` 문법은 Iceberg SQL 확장이 켜져 있어야 동작한다.

```javascript
spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions
```

## 1. Expire Snapshots

```sql
CALL my_lake.system.expire_snapshots(
    table       => 'log_db.user_event',
    older_than  => TIMESTAMP '2026-10-04 00:00:00',  -- 이보다 오래된 snapshot 삭제
    retain_last => 10                                -- 단, 최근 10개는 무조건 유지
);
```

- 만료된 snapshot에서만 참조하던 **데이터 파일과 manifest도 함께 지워진다.**
- 인자를 생략하면 테이블 속성이 기준이 된다.
	- `history.expire.max-snapshot-age-ms` (기본 5일)
	- `history.expire.min-snapshots-to-keep` (기본 1개)
- 만료된 시점으로는 **time travel이 불가능**해진다. 롤백이나 과거 조회가 필요한 기간보다 길게 잡아야 한다.

## 2. Remove old metadata files

```sql
ALTER TABLE my_lake.log_db.user_event SET TBLPROPERTIES (
    'write.metadata.delete-after-commit.enabled' = 'true',  -- 기본값 false
    'write.metadata.previous-versions-max'       = '100'    -- 남길 metadata.json 개수
);
```

- 커밋마다 metadata.json이 하나씩 새로 생긴다. 켜두면 지정 개수를 넘는 오래된 것부터 자동 삭제된다.
- **기본값이 꺼져 있다.** 스트리밍처럼 커밋이 잦은 테이블은 이걸 안 켜면 metadata 파일이 무한히 쌓인다.

## 3. Delete orphan files

```sql
-- 먼저 dry_run으로 무엇이 지워질지 확인
CALL my_lake.system.remove_orphan_files(
    table      => 'log_db.user_event',
    older_than => TIMESTAMP '2026-10-08 00:00:00',
    dry_run    => true
);
-- 결과 확인 후 dry_run 빼고 실행
```

- 실패한 쓰기 작업 등이 남긴, **어떤 snapshot도 참조하지 않는 파일**을 찾아 지운다.
- `older_than`은 **가장 긴 쓰기 작업 시간보다 넉넉하게**(기본 3일) 잡는다. 너무 짧으면 지금 쓰고 있는 중인 파일을 고아로 오인해 지워 데이터가 깨진다.
- 스토리지 전체를 나열하는 무거운 작업이라 주 1회 정도로 돌린다.

## Java API

```java
Table table = catalog.loadTable(TableIdentifier.of("log_db", "user_event"));
long cutoff = System.currentTimeMillis() - TimeUnit.DAYS.toMillis(7);

// Expire Snapshots (core API: 단일 프로세스에서 파일 삭제)
table.expireSnapshots()
     .expireOlderThan(cutoff)
     .retainLast(10)
     .commit();

// 대용량이면 Spark Actions로 삭제를 분산 처리
SparkActions.get()
    .expireSnapshots(table).expireOlderThan(cutoff).retainLast(10).execute();

// Orphan files는 Spark Actions 필요 (core API에 없음)
SparkActions.get()
    .deleteOrphanFiles(table)
    .olderThan(System.currentTimeMillis() - TimeUnit.DAYS.toMillis(3))
    .execute();
```
