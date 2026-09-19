# Spring for Kafka Example

> 작성일: 2026-01-04

```javascript
/* @KafkaListener 는 아래와 같은 일을 처리
- Kafka Consumer 생성
- poll loop 실행
- thread 관리
- error 처리
- message 변환
- retry / DLT
*/
@KafkaListener(id = "fooGroup", topics = "topic1", groupId = "foo-consumer-group")
public void listen(byte[] payload) {
    String message = new String(payload);
    logger.info("Received message: {}", message);

    if (message.contains("fail")) {
        throw new RuntimeException("forced failure");
    }
    // 실제 비즈니스 처리
}
// 예외 처리 (여기선 DLT)
@Bean
public CommonErrorHandler errorHandler(KafkaOperations<Object, Object> template) {
    return new DefaultErrorHandler(
        new DeadLetterPublishingRecoverer(
            template,(record, ex) -> new TopicPartition("topic1-dlt", record.partition())
        ),
        new FixedBackOff(1_000L, 2) // new FixedBackOff(0L, 0) 바로 전송
    );
}
```
