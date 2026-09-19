# Bucket

> 작성일: 2026-04-07

- 값을 해시해서 N개의 그룹으로 나누는 것

```javascript
CREATE TABLE my_lake.log_db.user_event (
    user_id BIGINT,
    event_time TIMESTAMP,
    event_type STRING,
    page STRING
)
USING iceberg
PARTITIONED BY (
    day(event_time),
    bucket(16, user_id)
);
1. event_time을 일 단위로 나눔 // pruning
2. user_id를 16개 bucket으로 나눔 // bucketing
>>> 예시 user_123 → hash → 123456 → 123456 % 16 = 3
```
