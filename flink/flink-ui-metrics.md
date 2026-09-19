# Flink UI Metrics

> 작성일: 2026-08-17

| Metric | 의미 | 언제 보는가 |
| --- | --- | --- |
| `numRecordsIn` | 들어온 누적 레코드 수 | 입력량 확인 |
| `numRecordsOut` | 내보낸 누적 레코드 수 | 처리/출력량 확인 |
| `numRecordsInPerSecond` | 초당 입력 건수 | 현재 처리량 |
| `numRecordsOutPerSecond` | 초당 출력 건수 | 현재 처리량 |
| `busyTimeMsPerSecond` | 1초 중 실제 처리에 바쁜 시간 | CPU/처리 병목 |
| `idleTimeMsPerSecond` | 1초 중 할 일이 없어 쉰 시간 | 입력 부족 여부 |
| `backPressuredTimeMsPerSecond` | 1초 중 downstream 때문에 막힌 시간 | 병목 Operator 찾기 |
