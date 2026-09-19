# MergeTree

> 작성일: 2026-02-22

- 로그/이벤트 이력 저장에 유리
- 데이터를 insert하면 그대로 쌓이고, 같은 key 데이터가 여러 번 들어와도 중복 제거 안 함
- MergeTree에서 merge는 데이터를 줄이는 게 아니라 파일을 정리하는 작업
	- 쓰기 → 정렬 → 병합
	- Background 병합 주기
		- 같은 Partition 안에 작은 Part가 많이 생겼을 때
		- Part 개수가 설정 임계치를 넘었을 때
		- TTL / DELETE / ReplacingMergeTree 정리가 필요할 때
	- Background 병합 히스토리 확인

```javascript
SELECT *
FROM system.part_log
WHERE event_type = 'MergeParts'
ORDER BY event_time DESC;
```
