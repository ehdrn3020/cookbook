# State Snapshots

> 작성일: 2026-05-27

참조 : [Fault Tolerance](https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/learn-flink/fault_tolerance/)

- Externalized Checkpoint
	- 사용자가 수동으로 복구 가능
	- 특정 checkpoint 기준 재시작 가능
- SavePoint
	- 사용자가 직접(trigger manually) 생성하는 snapshot
- Exactly Once Guarantees 최소 3개가 필요
	- Checkpoint 활성화 ( execution.checkpointing.mode: EXACTLY_ONCE )
	- Source가 replay 가능해야 함 ( ex. Kafka offset )
	- Sink가 transactional 또는 idempotent 해야 함 ( sink가 단순 insert면 X )
