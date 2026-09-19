# Garbage Collection 길어지는 원인

> 작성일: 2026-05-10

## 원인

- 메타데이터 폭증 ( NameNode는 메모리에 들고있는 정보 )
	- → NameNode heap 증설, 불필요한 파일 삭제, namespace/file count 모니터링
- replication 스케줄링 과다
	- → `dfs.namenode.replication.work.multiplier.per.iteration` 낮추기
	- → `dfs.namenode.replication.max-streams` 과도하게 올리지 않기
- small file 문제
	- → 작은 파일 병합, Spark/Hive output 파일 수 줄이기, Parquet/ORC compact 포맷 사용

## GC 길어지면 생기는 문제

- NameNode 멈춤 (치명적)
- heartbeat 끊김
- client timeout
