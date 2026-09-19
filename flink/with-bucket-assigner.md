# withBucketAssigner()

> 작성일: 2026-03-31

- FileSink가 각 레코드를 어떤 하위 디렉터리(bucket)에 쓸지 정하는 규칙을 주입하는 부분
- 반환된 bucket id의 `toString()` 값이 실제 경로명에 붙음

```plain text
FileSink<EnrichedEvent>sink_hdfs=FileSink
.forRowFormat(
	newPath(HDFS_OUTPUT_PATH), newSimpleStringEncoder<EnrichedEvent>("UTF-8")
)
.withBucketAssigner(
	newDateTimeBucketAssigner<>("'yyyy='yyyy/'mm='MM/'dd='dd/'hh='HH")
)
.withRollingPolicy(OnCheckpointRollingPolicy.build())
.build();

>>>
new Path(HDFS_OUTPUT_PATH) : 최상위 저장 경로
withBucketAssigner(...) : 그 아래에 어떤 서브 경로로 나눌지 결정
withRollingPolicy(...) : 파일을 언제 닫고 새 part file로 바꿀지 결정
```
