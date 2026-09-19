# Table API

> 작성일: 2026-05-04

- 예제 : [Real Time Reporting with the Table API](https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/try-flink/table_api/)

```javascript
// Flink 실행 환경 생성 (Streaming or Batch)
EnvironmentSettings settings 
	= EnvironmentSettings.inStreamingMode(); 
	//EnvironmentSettings.inBatchMode(); -- 스트리밍 로직을 "한 번 실행 후 종료" 방식
TableEnvironment tEnv = TableEnvironment.create(settings);

// 외부 시스템(Kafka, 파일 등)을 Flink Table로 추상화
tEnv.executeSql("CREATE TABLE transactions (...) WITH (...)");
```

- Window 적용 (Adding Windows)

```javascript
// Streaming에서 "언제 결과를 확정할지" 결정
.window(
    Tumble.over(lit(1).hour())
    .on($("transaction_time"))
    .as("log_ts")
)
```
