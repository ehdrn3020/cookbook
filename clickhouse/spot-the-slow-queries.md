# Spot The Slow Queries

> 작성일: 2026-04-15

- [Guide for query optimization | ClickHouse Docs](https://clickhouse.com/docs/optimize/query-optimization)

## Query logs

ClickHouse가 자동으로 기록하는 **쿼리 실행 로그 테이블 (`system.query_log`)**

```javascript
// 가장 오래 걸린 쿼리 10개 찾기
SELECT
    query,
    query_duration_ms,
    read_rows,
    read_bytes,
    memory_usage
FROM system.query_log
WHERE type = 'QueryFinish'
ORDER BY query_duration_ms DESC
LIMIT 10;
```

## Explain statement

쿼리 실행 계획 보기

```javascript
EXPLAIN
SELECT ...

EXPLAIN PIPELINE
SELECT ...
```

## system.settings

ClickHouse의 현재 설정값을 확인하는 시스템 테이블

```javascript
SELECT
    name,
    value,
    changed
FROM system.settings
WHERE name IN (
    'max_threads',
    'max_memory_usage',
    'max_execution_time',
    'max_bytes_before_external_group_by',
    'max_bytes_before_external_sort',
    'join_algorithm',
    'allow_experimental_analyzer'
);
```

## system.parts

- MergeTree 계열 테이블의 part 상태를 보는 시스템 테이블
- MergeTree는 데이터를 하나의 큰 파일로 저장하는 게 아니라, 여러 개의 **part**로 저장
	- 테이블별 총 크기, row 수, 날짜 범위, 일평균 증가량, 파티션별 part 개수 확인
