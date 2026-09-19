# GTID

> 작성일: 2025-11-10

Global Transaction ID

- 모든 트랜잭션에 전역 고유 ID를 붙여 복제/장애전환 시 "어디까지 적용했는지"를 **좌표(binlog 파일/포지션) 대신 GTID 집합**으로 추적하는 방식 → 페일오버/재동기화가 단순·안전

## MariaDB GTID 설정

- show global variables like '%GTID%';
- gtid_strict_mode=ON ( GTID 모드 사용 )
- log_slave_updates=ON ( Slave 서버에도 GTID 모드 사용 )

## GTID 구성 예시

- domain_id - server_id - sequence_number
- 확인 쿼리

```javascript
SHOW VARIABLES LIKE 'gtid%';
SHOW VARIABLES LIKE 'server_id';
```

- my.cnf 설정 예시

```javascript
[mysqld]
server_id = 4001
gtid_domain_id = 40001
binlog_format = ROW
log_bin = ON
log_slave_updates = ON
gtid_strict_mode = ON
expire_logs_days=7
binlog_row_image=FULL
# binlog_rows_query_log_events=ON # query의 원본도 확인
```

- 소스 DB의 바이너리 로그(binlog) 진행 좌표 확인
	- SHOW MASTER STATUS;

## 테스트

- connector delete -> 트랜잭션 N발생 → connector start : 초기 스냅샷으로 데이터 누락 발생
- connector pause -> 트랜잭션 N발생 → connector resume : 이전메세지까지 모두 복구
