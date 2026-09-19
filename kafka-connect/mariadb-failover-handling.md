# Mariadb Failover 조치

> 작성일: 2025-12-13

- GTID 값 MariaDB에서 확인

```javascript
SHOW BINARY LOGS;
SHOW BINLOG EVENTS IN 'ON.000002' FROM 1971 LIMIT 20;
```

- 방법1 : 오프셋 초기화하고 처음부터 다시 구성
- 방법2: 기존 토픽/오프셋 유지하면서 누락없이 CDC

- connector json "name"을 바꾸면 새로운 Connector로 인식 ( SUFFIX "_v2" 요런식으로 )
	- Connector offset / task 상태는 초기화
		- 기존 binlog 위치 이어받지 않음
	- Kafka 토픽과 Debezium schema history는 초기화되지 않음
