# Distributed Cluster

> 작성일: 2026-03-18

## 정의

- Distributed(cluster, db, table, sharding_key) 는 ClickHouse에서 분산 테이블 엔진
- 즉, 실제 데이터는 각 샤드/리플리카의 로컬 테이블에 있고, Distributed 테이블은 그것들을 하나의 논리 테이블처럼 조회/쓰기 위한 라우터 역할
- [Cluster Deployment | ClickHouse Docs](https://clickhouse.com/docs/architecture/cluster-deployment)

## 구조

- 총 서버: **4대 / 2 shard × 2 replica**

```javascript
                    [ Client / App ]
                            │
                            │
                            ▼
                [ Distributed Table ]
                 (events_dist)
                            │
        ┌───────────────────┴───────────────────┐
        │                                       │
    ┌───────────────┐                   ┌───────────────┐
    │   SHARD 1     │                   │   SHARD 2     │
    │ (data group A)│                   │ (data group B)│
    └───────┬───────┘                   └───────┬───────┘
            │                                   │
    ┌───────┴────────┐                  ┌───────┴────────┐
    │                │                  │                │
┌─────────────┐  ┌─────────────┐   ┌─────────────┐  ┌─────────────┐
│ ch-node-01  │  │ ch-node-02  │   │ ch-node-03  │  │ ch-node-04  │
│ replica 1   │  │ replica 2   │   │ replica 1   │  │ replica 2   │
│ shard 1     │  │ shard 1     │   │ shard 2     │  │ shard 2     │
└─────────────┘  └─────────────┘   └─────────────┘  └─────────────┘
```

## config.xml

```javascript
<clickhouse>
    <remote_servers>
        <my_cluster>
            <secret>my_shared_secret</secret>

            <shard>
                <name>shard_01</name>
                <weight>1</weight>
                <internal_replication>true</internal_replication>
                <replica>
                    <name>ch-node-01</name>
                    <host>ch-node-01</host>
                    <port>9000</port>
                    <user>default</user>
                    <password></password>
                    <priority>1</priority>
                </replica>
                <replica>
                    <name>ch-node-02</name>
                    <host>ch-node-02</host>
                    <port>9000</port>
                    <user>default</user>
                    <password></password>
                    <priority>1</priority>
                </replica>
            </shard>

            <shard>
                <name>shard_02</name>
                <weight>1</weight>
                <internal_replication>true</internal_replication>
                <replica>
                    <name>ch-node-03</name>
                    <host>ch-node-03</host>
                    <port>9000</port>
                    <user>default</user>
                    <password></password>
                    <priority>1</priority>
                </replica>
                <replica>
                    <name>ch-node-04</name>
                    <host>ch-node-04</host>
                    <port>9000</port>
                    <user>default</user>
                    <password></password>
                    <priority>1</priority>
                </replica>
            </shard>
        </my_cluster>
    </remote_servers>
</clickhouse>
```

## 테이블 생성

```javascript
- event_all 에 조회하면 각 샤드의 event_local 을 모아서 보여줌
- sharding_key(user_id) : INSERT 할 때 어느 샤드로 보낼지 결정하는 식(expression)

CREATE TABLE analytics_db.event_local
(
    event_time DateTime,
    user_id UInt64,
    value UInt32
)
ENGINE = MergeTree
ORDER BY (event_time, user_id);

CREATE TABLE analytics.event_all
AS analytics.event_local
ENGINE = Distributed(my_cluster, analytics_db, event_local, user_id);
```
