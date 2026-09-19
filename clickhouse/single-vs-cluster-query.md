# Single / Cluster Query 차이

> 작성일: 2026-05-15

## 요약

- 싱글 = MergeTree table
- 클러스터 = Replicated local table + Distributed table(ON CLUSTER + sharding key)

## 테이블 생성

```javascript
// 클러스터에서는 보통 local table을 각 노드에 만들고
CREATE TABLE log_db.silver_event_local
ON CLUSTER cluster_1
(
    event_time DateTime,
    broad_no UInt64,
    log_name String
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/log_db/silver_event_local',
    '{replica}'
)
PARTITION BY toYYYYMMDD(event_time)
ORDER BY (broad_no, event_time);

// 그 위에 Distributed table을 만든다.
CREATE TABLE log_db.silver_event
ON CLUSTER cluster_1
AS log_db.silver_event_local
ENGINE = Distributed(
    cluster_1,
    log_db,
    silver_event_local,
    broad_no
);
```

## MATERIALIZED VIEW 생성

```javascript
// 노드 3개면 각 노드의 local table마다 MV가 있어야 한다.
// 보통은 ON CLUSTER로 한 번에 생성한다.
CREATE MATERIALIZED VIEW log_db.mv_gold_live_broad_1m_local
ON CLUSTER cluster_1
TO log_db.gold_live_broad_1m       // 로컬 테이블면 중복 데이터가 생성 됨
AS
SELECT
    toStartOfMinute(event_time) AS bucket_minute,
    count() AS broad_count
FROM log_db.silver_live_broad_event_local
GROUP BY bucket_minute;
```

```plain text
싱글 = MergeTree 하나
클러스터 = Replicated local table + Distributed table + ON CLUSTER + sharding key
```

- MV는 특히 조심. 보통은 Distributed → Distributed로 MV를 만들기보다 local → local MV를 만들고, 조회만 Distributed로 하는 구조가 안전
