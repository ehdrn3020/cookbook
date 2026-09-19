# ClickHouse Sink

> 작성일: 2026-01-22

- JDBC Sink를 사용하면 성능 한계
	- JDBC Sink는 행 기반 + DB 커밋 중심 구조라 ClickHouse의 컬럼 '지향 + 대량 블록 입력' 과 맞지 않아 효율성이 떨어짐
	- 그래서 http sink 사용
- Exactly-once가 안 되는 이유
	- ClickHouse는 '트랜잭션 롤백과 체크포인트 연동 커밋'이 불가능한 DB
	- 해결 방안 : 동일 PK → 최신 row만 유지, 그래서 중복 허용 → 결과적 Exactly-once
