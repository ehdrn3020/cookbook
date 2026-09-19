# MSSQL LSN

> 작성일: 2025-10-26

- LSN(Log Sequence Number)의 SQL Server는 DB에 발생하는 모든 변경 사항(INSERT, UPDATE, DELETE 및 스키마 변경)을 트랜잭션 로그 파일에 기록하며, LSN은 이 로그 내에서 특정 시점의 위치를 나타 냄
- connect-offsets이 LSN을 읽어서 해당 지점부터 CDC Stream을 읽음, 재시작 후에도 데이터 누락 안함
