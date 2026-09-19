# State Backends

> 작성일: 2026-05-27

참조 : [Fault Tolerance](https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/learn-flink/fault_tolerance/) : 상태(key/value state)를 어디에 저장하고, 스냅샷(checkpoint/savepoint)을 뜨는가

## RocksDB 기반 State Backend

- 상태를 로컬 디스크(tmp dir)에 저장
- 메모리보다 훨씬 큰 상태 저장 가능
- 직렬화/역직렬화 필요 → 느림
- Full / Incremental Snapshot 가능
- 대용량 state에 적합
- 작업 중 state 위치 : TaskManager 로컬 디스크 RocksDB 디렉터리
- flink-conf.yaml

```javascript
state.backend: rocksdb
state.checkpoints.dir: hdfs:///flink/checkpoints
execution.checkpointing.interval: 60s
state.backend.incremental: true        // incremental checkpoint
```

## Heap 기반 State Backend

- 상태를 Java Heap 메모리에 저장
- 매우 빠름
- Heap 많이 필요
- GC 영향 받음
- Full Snapshot만 가능
- 작업 중 state 위치 : TaskManager JVM Heap 메모리
- flink-conf.yaml

```javascript
state.backend: hashmap
state.checkpoints.dir: hdfs:///flink/checkpoints
execution.checkpointing.interval: 60s
```
