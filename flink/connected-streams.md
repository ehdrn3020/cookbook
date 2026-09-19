# Connected Streams

> 작성일: 2026-02-04

참조 : [Data Pipelines & ETL](https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/learn-flink/etl/)

- 컨트롤 스트림이 데이터 스트림의 동작을 실시간으로 제어

```javascript
DataStream<String> control = env
    .fromData("DROP", "IGNORE")
    .keyBy(x -> x);

DataStream<String> streamOfWords = env
    .fromData("Apache", "DROP", "Flink", "IGNORE")
    .keyBy(x -> x);

control
    .connect(streamOfWords)
    .flatMap(new ControlFunction())
    .print();
/*
Apache   → 정상
DROP     → 차단 대상
Flink    → 정상
IGNORE   → 차단 대상
*/
```
