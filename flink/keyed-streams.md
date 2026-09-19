# Keyed Streams

> 작성일: 2026-02-04

참조 : [Data Pipelines & ETL](https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/learn-flink/etl/)

- `keyBy()`는 같은 키를 가진 이벤트들을 "같은 작업자(Task)"로 모으며, 네트워크 셔플을 일으킴
- 데이터를 어떤 Subtask로 보낼지 재분배하는 기능
- 암묵적 State : .key만 사용으로도 이미 stateful streaming
- .window() 와 같이 사용, window = state의 수명 제한
	- unbounded key space 는 무한이 state가 증식됨
	- .window(TumblingEventTimeWindows.of(Time.minutes(10)))
		- 10분 후 윈도우가 끝나면 state 자동 정리

## Stateless Transformations

- `map()` 과 `flatmap()` 의 차이
	- DataStream<T>.map(...)
		- 1 → 1 변환 (무조건 하나 출력)

		```javascript
		DataStream<EnrichedRide> enrichedNYCRides = rides
		    .filter(new RideCleansingSolution.NYCFilter())
		    .map(new Enrichment());
		```

	- DataStream<T>.flatMap(...)
		- 0 ~ N 변환 (출력 개수 자유)

		```javascript
		DataStream<EnrichedRide> enrichedNYCRides = rides
		    .flatMap(new NYCEnrichment());
		```
