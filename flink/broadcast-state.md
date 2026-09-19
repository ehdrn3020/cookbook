# Broadcast State

> 작성일: 2026-09-19

Context 객체

- 작은 기준/설정 데이터를 모든 병렬 Task에 뿌려서 각 Task가 동일하게 참조하도록 하는 State
- Map 형태로 저장된다. ( rule1 → 10, rule2 → 200 ), `MapStateDescriptor`를 사용해 관리
- Broadcast Stream과 일반 Stream을 동시에 입력으로 받는 특정 Operator에서만 사용
- 하나의 Operator가 여러 개의 Broadcast State를 가질 수 있다.
	- [Broadcast State :: Apache Flink](https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/datastream/fault-tolerance/state/#broadcast-state)

- 일반 Stream이 `keyBy()` 되어 있으면: **KeyedBroadcastProcessFunction**
- 일반 Stream이 `keyBy()` 되어 있지 않으면 **BroadcastProcessFunction** 사용함.
	- [The Broadcast State Pattern :: Apache Flink](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/fault-tolerance/broadcast_state/)

## 예제 코드

- processBroadcastElement() → Broadcast State 읽기 + 쓰기 가능
- processElement() → Broadcast State 읽기만 가능

```java
public class BroadcastStateExample {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env =
                StreamExecutionEnvironment.getExecutionEnvironment();

        // 1. 일반 Stream
        DataStream<StreamerEvent> streamerStream = ...;
        
        // 2. Rule Stream
        DataStream<AchievementRule> ruleStream = ...;

        // 3. Broadcast State 정의
        MapStateDescriptor<String, Integer> ruleStateDescriptor =
                new MapStateDescriptor<>(
                        "achievement-rules",
                        String.class,
                        Integer.class
                );

        // 4. Rule Stream broadcast
        BroadcastStream<AchievementRule> broadcastRules =
                ruleStream.broadcast(ruleStateDescriptor);

        // 5. 일반 Stream keyBy
        KeyedStream<StreamerEvent, String> keyedStream =
                streamerStream.keyBy(StreamerEvent::getStreamerId);

        // 6. 두 Stream 연결
        keyedStream
                .connect(broadcastRules)
                .process(
                        new KeyedBroadcastProcessFunction<
                                String,
                                StreamerEvent,
                                AchievementRule,
                                String>() {

                            @Override
                            public void processBroadcastElement(
                                    AchievementRule rule,
                                    Context ctx,
                                    Collector<String> out) throws Exception {

                                ctx.getBroadcastState(ruleStateDescriptor)
                                        .put(rule.getName(), rule.getValue());
                            }

                            @Override
                            public void processElement(
                                    StreamerEvent event,
                                    ReadOnlyContext ctx,
                                    Collector<String> out) throws Exception {

                                Integer level =
                                        ctx.getBroadcastState(ruleStateDescriptor)
                                                .get("LIVE_DAY_LEVEL_1");

                                if (event.getTotalBroadcastDays() >= level) {
                                    out.collect(event.getStreamerId() + " 달성");
                                }
                            }
                        });

        env.execute();
    }
}
```
