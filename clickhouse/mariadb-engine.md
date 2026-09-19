# MariaDB Engine

> 작성일: 2026-05-14

- `ENGINE = MySQL` 테이블은 ClickHouse에 데이터를 저장하지 않음
- 조회 시점마다 MariaDB 원본 테이블을 직접 조회 ( 리소스 주의 )
- ClickHouse에 최신 상태를 저장하기 위해 Materialized View를 따로 생성해서 저장해야 함

```javascript
// MariaDB Engine Table Create
CREATE TABLE IF NOT EXISTS log_db.mariadb_user_info_tbl (
	`user_id` String,
  `user_nick` String,
  `gender` String
)
ENGINE = MySQL(
    '123.123.123.123:3306',
    'user_db',
    'user_info_tbl',
    'myuser',
    'mypassword'
);

// Materialized Views
CREATE MATERIALIZED VIEW IF NOT EXISTS log_db.mv_mariadb_user_info_state
REFRESH EVERY 1 MINUTE
APPEND TO log_db.mariadb_user_info_state
AS
SELECT
  toStartOfMinute(now()) AS bucket_time,
  user_id,
  user_nick,
  gender
FROM log_db.mariadb_user_info_tbl ;  
    
// State Table
CREATE TABLE IF NOT EXISTS log_db.mariadb_user_info_state (
	`bucket_time` DateTime,
	`user_id` String,
  `user_nick` String,
  `gender` String
)
ENGINE = ReplacingMergeTree(bucket_time)
ORDER BY user_id;

// 조회
SELECT count(*) 
FROM log_db.mariadb_user_info_state
WHERE bucket_time = '2026-05-14 10:50:00';
```
