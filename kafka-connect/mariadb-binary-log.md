# MariaDB Binary Log

> 작성일: 2025-11-09

CDC 설정된 후 관련 파일 : /var/lib/mysql

- 바이너리 로그 파일 ( ON.000001 ) : DB에서 일어난 변경 이벤트(DML, DDL 등)가 순서대로 기록된 실제 데이터 파일
- 바이너리 로그 인덱스 파일 ( ON.index ) : 유효한 binlog 파일들의 경로 목록
- binlog 보존 기간이 짧으면 kafka cdc 에서 읽을 수 없음
- binlog_format=ROW 으로 사용해야 Debezium기준 바이너리 로그 파일 읽을 수 있음
	- binlog_format=STATEMENT : **실행한 SQL문 자체**를 binlog에 기록
	- binlog_format=ROW : **실제로 변경된 각 행의 값**을 기록
	- binlog_format=MIXED : 상황에 따라 STATEMENT, ROW 선택
- MariaDB binlog → connect-offset 에 읽은 위치까지 저장
