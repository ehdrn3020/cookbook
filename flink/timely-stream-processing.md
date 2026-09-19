# Timely Stream Processing

> 작성일: 2026-07-31

## Processing Time

이벤트를 실제로 처리한 서버의 현재 시간

- 별도의 timestamp 추출이나 Watermark 설정이 필요하지 않음
- 데이터를 다시 처리하더라도 다음과 같이 결과가 달라질 수 있음
	- Kafka 적체, 네트워크 지연, Backpressure, 장애, Operator 처리 속도 이슈에 따라

## Event Time

이벤트가 실제로 발생한 시간

- 이벤트가 순서대로 도착한다는 보장이 없어서 WaterMark가 필요
- 도착 순서와 관계없이 이벤트 발생 시간 기준으로 묶이므로 결과가 더 정확하고 일관적
- 늦게 도착하는 이벤트를 기다려야 하므로 Processing Time보다 결과가 늦게 나올 수 있음

## Lateness란?

Watermark가 선언한 조건을 어기고 **뒤늦게 도착한 이벤트**를 의미합니다.
