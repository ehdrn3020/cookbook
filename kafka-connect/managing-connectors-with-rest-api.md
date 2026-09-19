# REST API로 커넥터 관리

> 작성일: 2025-11-16

- 커넥터
	- 상태 : `curl -s localhost:8083/connectors/my_connect/status`
	- 토픽 : `curl -s localhost:8083/connectors/my_connect/topics`
	- 설정 : `curl -s localhost:8083/connectors/my_connect/config`
	- 오프셋 : `curl -s localhost:8083/connectors/my_connect/offsets`
- 태스크
	- `curl -s localhost:8083/connectors/my_connect/tasks`
	- `curl -s localhost:8083/connectors/my_connect/tasks/0/status | jq`
- 로깅 : `curl -s localhost:8083/admin/loggers`
