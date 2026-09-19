# LowCardinality

> 작성일: 2026-02-13

- 문자열 컬럼을 "사전 + 정수키"(Dictionary encoding) 형태로 저장
- 같은 값이 많이 반복될 때 메모리/디스크를 줄이고 GROUP BY 같은 연산을 빠르게 만드는 타입

```javascript
CREATE TABLE lc_demo
(
  ts DateTime,
  country LowCardinality(String),
)
ENGINE = MergeTree
-- country 는 KR, US, ER일때 [1,2,3] 이렇게 저장되며 SELECT 시에 맵핑 후 String으로 보임
```
