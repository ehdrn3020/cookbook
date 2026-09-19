# Query Execution Plan Debugging

> 작성일: 2026-05-08

- 실행 계획을 확인 할 수 있음 : `df.explain("formatted")`

## Job & Stage Example

```plain text
val df=spark.read.parquet("/data/input")

val result=df
  .filter($"status"==="OK")
  .groupBy($"user_id")
  .count()

result.write.parquet("/data/output")
```

- Job은 **Action**이 실행될 때 Job이 생깁니다.

```plain text
// Action
result.write.parquet("/data/output")
--> Job 1개 생성
```

- Stage는 Shuffle 기준으로 나뉩니다.

```plain text
// 위 코드에서 shuffle이 발생하는 부분
.groupBy($"user_id")
--> user_id 기준으로 데이터를 다시 모아야 하므로 네트워크 셔플이 발생합니다.
```

- Job 하나 안에서 Stage가 나뉩니다.

```plain text
Job 0
 ├─ Stage 0: parquet read + filter
 └─ Stage 1: groupBy count + parquet write
```
