# Flyway - Schema migration tools

> 작성일: 2026-05-02

- 관련 툴 참조 : [Schema migration tools for ClickHouse | ClickHouse Docs](https://clickhouse.com/docs/ko/knowledgebase/schema_migration_tools)

## REFRESH EVERY 1 MINUTE

- 주기적으로 Time Interval에 따라 계산하는 Materialized View를 만들 때

```javascript
CREATE MATERIALIZED VIEW mv
REFRESH EVERY 1 MINUTE
APPEND TO target
AS SELECT ...

즉 
INSERT → MV 실행 ❌
시간 → MV 실행 ⭕
```

## APPEND vs REPLACE (핵심 차이)

- APPEND
	- 실행될 때마다 결과를 계속 쌓음
	- 기존 데이터 유지

```plain text
10:14 / SST / 10
10:15 / SST / 12
10:16 / SST / 8
```

- REPLACE
	- 기존 데이터 덮어씀
	- 최신 상태 유지
