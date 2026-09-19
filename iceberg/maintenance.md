# Maintenance

> 작성일: 2026-04-14

- Iceberg는 데이터를 append하고 snapshot을 계속 쌓는 구조라서, 오래된 snapshot / metadata / 안 쓰는 파일을 주기적으로 청소
- append-only + snapshot 기반
- Expire Snapshots / Remove old metadata files / Delete orphan files 주기적 실행 필요
- Expire Snapshots : Java API로 실행 가능 / Spark SQL 프로시저로도 가능
