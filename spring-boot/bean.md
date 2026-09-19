# @Bean

> 작성일: 2026-01-06

- @Bean은 프레임워크가 제공하는 기본 동작 대신 사용자가 정의한 구현체를 주입하기 위한 확장 포인트
- @Autowired 어노테이션을 사용해, Bean에 등록된 객체를 사용
- @Bean은 의존성 주입의 기준은 함수명이 아닌 "타입" ( 같은 타임 여러 함수명이면 우선순위로 구분 )

## KafkaTemplate

```javascript
/* Spring Boot 
Foo1은 Producer에서 Kafka로 보낼 때 "직렬화"되고,
Consumer가 Kafka에서 받을 때 "역직렬화"된다.
*/
import org.springframework.kafka.core.KafkaTemplate;
@Autowired
private KafkaTemplate<Object, Object> template;
this.template.send("foos", new Foo1(what));

->
/* Only Java */
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

// 2. KafkaProducer 생성 (Spring @Autowired 대체)
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
