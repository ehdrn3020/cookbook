# Spring for Kafka

> 작성일: 2026-01-06

- 참조 : [Spring for Apache Kafka](https://docs.spring.io/spring-kafka/reference/)
- 참조 : [Messaging - Apache Kafka (Spring Boot)](https://docs.spring.io/spring-boot/reference/messaging/kafka.html)

- 메세지 기반 POJO(Message-driven POJO) 사용
	- POJO 란 비즈니스 로직 중심 / 단순한 필드 + 메서드 순수 자바 클래스
	- 코드 안에서는 "KafkaConsumer", "ConsumerRecord" 알 수 없고, "비즈니스 로직"만 작성
- 보내는 쪽은 `KafkaTemplate`, 받는 쪽은 `@KafkaListener`를 쓴다. 둘 다 Spring이 빈으로 만들어 준다.

## 의존성

```groovy
// build.gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.kafka:spring-kafka'   // 버전은 Boot BOM이 관리

    testImplementation 'org.springframework.kafka:spring-kafka-test'
}
```

- `spring-kafka`만 추가하면 `KafkaAutoConfiguration`이 동작해 `KafkaTemplate`, `ProducerFactory`, `ConsumerFactory`, 리스너 컨테이너가 자동으로 등록된다.
- 버전을 명시하지 않는 이유는 Spring Boot BOM이 호환되는 버전을 잡아주기 때문이다. 직접 올리면 Boot 버전과 어긋날 수 있다.

## 설정

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092

    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all                  # 리더+ISR 전부 받아야 성공 처리
      retries: 3

    consumer:
      group-id: foo-consumer-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.example.dto"   # 역직렬화 허용 패키지
```

- `spring.json.trusted.packages`를 안 잡으면 `JsonDeserializer`가 보안상 역직렬화를 거부한다. 운영에서 자주 걸리는 지점이다.

## 보내기 — KafkaTemplate

```java
import org.springframework.kafka.core.KafkaTemplate;

@Service
public class FooProducer {

    private final KafkaTemplate<String, Object> template;

    // 생성자 주입 권장 (@Autowired 필드 주입도 동작은 한다)
    public FooProducer(KafkaTemplate<String, Object> template) {
        this.template = template;
    }

    public void send(String what) {
        // Foo1은 Producer에서 Kafka로 보낼 때 "직렬화"되고,
        // Consumer가 Kafka에서 받을 때 "역직렬화"된다.
        this.template.send("foos", new Foo1(what));
    }
}
```

- `send()`는 **비동기**다. 호출 즉시 리턴하고 실제 전송은 백그라운드에서 일어난다. 결과를 확인하려면 반환된 future를 쓴다.

```java
template.send("foos", key, new Foo1(what))
    .whenComplete((result, ex) -> {
        if (ex != null) {
            log.error("전송 실패 key={}", key, ex);
        } else {
            RecordMetadata meta = result.getRecordMetadata();
            log.info("전송 성공 partition={} offset={}", meta.partition(), meta.offset());
        }
    });
```

- 같은 key로 보내면 같은 파티션으로 가서 **key 단위 순서가 보장**된다. key를 안 주면 파티션에 분산되어 순서 보장이 없다.

### 설정을 코드로 잡고 싶을 때

- `application.yml`로 안 되는 세밀한 설정이 필요하면 `ProducerFactory`와 `KafkaTemplate`을 직접 `@Bean`으로 등록한다. 자동 설정은 `@ConditionalOnMissingBean`이라 물러난다.

```java
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
        return new DefaultKafkaProducerFactory<>(props);
    }

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate(ProducerFactory<String, Object> pf) {
        return new KafkaTemplate<>(pf);
    }
}
```

## Spring 없이 순수 Java로 하면

- 위 `template.send(...)` 한 줄이 대신해 주는 일을 직접 쓰면 이만큼이 된다. 설정·직렬화·생명주기를 전부 손으로 관리해야 한다.

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;
import org.apache.kafka.common.serialization.ByteArraySerializer;

// 1. Kafka Producer 설정
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
          StringSerializer.class.getName());
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
          ByteArraySerializer.class.getName());

// 2. KafkaProducer 생성 (Spring 주입 대체)
KafkaProducer<String, byte[]> producer = new KafkaProducer<>(props);

// 3. Foo1 → byte[] 직렬화 (Spring JsonSerializer가 하던 일)
Foo1 foo = new Foo1("hello");
byte[] valueBytes = foo.getWhat().getBytes();

// 4. ProducerRecord 생성
ProducerRecord<String, byte[]> record =
        new ProducerRecord<>("foos", valueBytes);

// 5. 메시지 전송
producer.send(record);

// 6. 종료 처리
producer.flush();
producer.close();
```

- 차이는 코드량보다 **생명주기**다. 순수 Java는 `close()`를 직접 불러야 하지만, 빈으로 등록하면 컨테이너 종료 시 Spring이 정리한다.

## 받기 — @KafkaListener

```java
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
```

- 메서드 파라미터 타입만 바꾸면 Spring이 변환해 준다. `byte[]` 대신 DTO를 받으면 `JsonDeserializer`가 역직렬화한다.
- `concurrency` 속성으로 리스너 스레드 수를 올릴 수 있다. **파티션 수를 넘으면 남는 스레드는 논다.**

```java
@KafkaListener(topics = "topic1", concurrency = "3")
```

## 에러 처리 — DLT

- 리스너에서 예외가 나면 기본은 재시도 후 로그만 남는다. 실패 메시지를 따로 모으려면 DLT(Dead Letter Topic)로 보낸다.

```java
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

- `FixedBackOff(interval, maxAttempts)` : 위 설정은 1초 간격으로 2회 재시도 후 DLT로 보낸다.
- 재시도 중에는 **해당 파티션의 처리가 멈춘다.** 간격을 길게 잡으면 그만큼 적체되므로 주의한다.

## 관련 문서

- [@Bean](bean.md) — 자동 설정 back off, 빈 등록 규칙
- [Datasource와 DSLContext](datasource_dslcontext.md) — 같은 패턴으로 DataSource를 직접 등록하는 경우
