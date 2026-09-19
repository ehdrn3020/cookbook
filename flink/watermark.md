# Water Mark

> 작성일: 2026-01-26

참조 : [Streaming Analytics](https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/learn-flink/streaming_analytics/)

- 윈도우 종료 시점을 결정
	- 윈도우 종류 텀블링, 슬라이딩, 세션 등등 같이 워터마크랑 사용됨
- 지연 데이터 허용 범위 표현
- 프로세스

```javascript
이벤트 시간 축 →
|----10:00----10:05----10:10----|

Watermark = 10:05
→ 10:05 이전 이벤트는 다 왔다고 판단
→ [10:00 ~ 10:05) 윈도우 닫힘

만약 10:03 이벤트가 10:06에 도착 하면? → late event(지연 이벤트)
```

- late event 처리 프로세스
	- ① 기본 그냥 버려짐
	- ② 허용 가능
	- ③ 따로 수집
- 워터마크 예제
	- 5초까지 허용

```javascript
DataStream<Event> stream = ...;

WatermarkStrategy<Event> strategy = WatermarkStrategy
        .<Event>forBoundedOutOfOrderness(Duration.ofSeconds(5))
        .withTimestampAssigner((event, timestamp) -> event.timestamp);

DataStream<Event> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(strategy);
```

- water mark를 설정 안하면 → 그냥 윈도우가 영원히 열린 채로 멈춤, emit 안함
- water mark = 0 으로 설정하면 → 늦게 온 데이터는 대부분 "늦은 이벤트(late event)"가 되어 버려지거나(side output), 새로운 윈도우로 들어가지 않음 (정책에 따라 처리)
- Bounded Unbounded Stream 모두 "Stream"으로 처리
	- Bounded Stream - 입력 데이터의 끝(end)이 명확히 존재하는 스트림
	- Unbounded Stream - 이론적으로 끝이 없고, 계속 들어오는 스트림
