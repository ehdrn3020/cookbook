# Hidden Partitioning

> 작성일: 2026-04-07

- Hive와 다르게 파티션이 컬럼으로 존재할 필요가 없음
- Partition 컬럼 신경을 사용자가 쓰지 않아도 됨

```javascript
CREATE TABLE logs (
  id bigint,
  broad_no bigint,
  event_time timestamp   ← ✅ 실제 컬럼
)
PARTITIONED BY (
  days(event_time),      ← ❌ 컬럼 아님 (transform)
  bucket(32, broad_no)  ← ❌ 컬럼 아님 (transform)
)

// 실행 쿼리
SELECT * FROM logs WHERE event_time >= '2026-01-13'
→ event_time → days(event_time) 변환
→ partition = 2026-01-13
→ 해당 파일만 읽음
→ BUT, partition = bucket(user_id) 이렇게 되어있으면 모든 파일을 읽음
```
