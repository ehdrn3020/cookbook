# Index Granule

> 작성일: 2026-01-21

- row 묶음 단위
- 8192 row 중 1개라도 걸리면 그 granule 전체를 읽음
	- index_granularity 를 줄이면 더 빨라질까?
		- 조건에 따라 다름
		- 인덱스 크기와 메모리가 증가
		- WHERE 조건이 Primary Key 앞부분 컬럼에 있고 그 조건의 선택도가 매우 높을 때
