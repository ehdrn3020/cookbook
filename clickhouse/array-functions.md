# Array관련 함수

> 작성일: 2026-02-15

```javascript
groupArray : 그룹별로 값들을 배열로
SELECT
  number % 2 AS g,
  groupArray(number) AS arr
FROM numbers(6)   -- 0,1,2,3,4,5
GROUP BY g
ORDER BY g;
-- 
| g | arr     |
| 0 | [0,2,4] |
| 1 | [1,3,5] |

arrayFilter : 배열에서 조건을 만족하는 원소만 남김
SELECT arrayFilter(x -> x > 2, [1,2,3,4]) AS filtered;
-- [3,4]

arrayJoin: 배열을 행으로 펼침
SELECT arrayJoin([10,20,30]) AS x;
-- rows: 10 / 20 / 30 

arrayMap: 배열의 각 요소에 람다 함수를 적용해서 새로운 배열을 만드는 함수
WITH arrayMap(
    d -> if(d >= 30, 'DELAYED', if(d >= 15, 'WARNING', 'ON-TIME')),
    groupArray(DepDelayMinutes)
) AS statuses
--- ['ON-TIME', 'ON-TIME', 'WARNING', 'DELAYED']

arrayFlatten : 중첩배열 1차원으로 평탄화
SELECT arrayFlatten([[1,2],[3],[4,5]]) AS flat;
-- [1,2,3,4,5]

arrayCompact : 연속으로 붙어있는 중복만 제거
SELECT arrayCompact([1,1,2,2,2,1]) AS compacted;
-- [1,2,1]

arrayDistinct : 배열 전체에서 중복 제거(순서는 첫 등장 기준)
SELECT arrayDistinct([3,1,3,2,1]) AS uniq;
-- [3,1,2]
```

## ClickHouse에서 Parquet 조회

- Array Join Column

```javascript
SELECT *
FROM file('data.parquet', ParquetMetadata);
>>>
row_groups Nested(
    num_rows UInt64,
    columns Nested(
        path String,
        total_compressed_size UInt64,
        statistics Nested(...)
    )
)

//그래서 Array Join 필요
SELECT
    row_group.num_rows
FROM file('data.parquet', ParquetMetadata)
ARRAY JOIN row_groups AS row_group;
>>>
num_rows
--------
100000
100000
```
