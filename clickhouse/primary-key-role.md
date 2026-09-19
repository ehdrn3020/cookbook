# Primary Key의 역할

> 작성일: 2026-01-21

- 쿼리를 빠르게 하기 위한 데이터 정렬 기준 ( = Order by )
- Unique 하지 않아도 됨
- **index granule (기본 8192 rows)** 단위로 min/max 값만 저장
- where 조건의 앞에 두어 쿼리 빠르게 함
