# SavePoint & CheckPoint

> 작성일: 2026-08-16

## SavePoint

```javascript
bin/flink list
bin/flink savepoint ${jobId}

# savepoint 저장
bin/flink savepoint 1b0f0cfd5359fe1bb76c35e6950d5ee1 hdfs://flinkmaster:8020/flink/savepoints/

# flink job 중지
bin/flink cancel d2742dedb0976c29861d9729d36e9bdc -s hdfs://flinkmaster:8020/flink/savepoints/savepoint-d2742d-aefc9706cc6d

# flink 시작, savepoint 지점부터
bin/flink run -s hdfs://flinkmaster:8020/flink/savepoints/savepoint-d2742d-aefc9706cc6d myjob.jar

# flink savepoint 저장과 동시에 job 중지
bin/flink stop --savepointPath hdfs://flinkmaster:8020/flink/savepoints --drain <jobId>
```

- Graceful Stop

```javascript
bin/flink stop <JOB_ID>

기본 Savepoint 경로가 설정돼 있다면
( state.savepoints.dir: hdfs://namenode:8020/flink/savepoints )
stop 명령만으로 해당 위치에 Savepoint를 만든 뒤 Job을 종료합니다
```

## CheckPoint

```javascript
execution.checkpointing.interval: 5 min
execution.checkpointing.mode: EXACTLY_ONCE
execution.checkpointing.dir: hdfs://namenode:8020/flink/checkpoints
```

## 복구 정책

- 세이브포인트가 존재하는 경우 해당 세이브포인트에서 실행되도록 설정합니다.
	- 정상적으로 종료된 경우
	- Job을 재배포 상황
- 체크포인트가 존재하는 경우 가장 최근 체크포인트에서 실행합니다.
	- 비정상적으로 종료된 경우 ( TaskManager 죽거나, 서버가 죽거나 )
	- **submit job 에서 savepoint 인자를 checkpoint meta data 경로로 주입**
		- 예시 : hdfs://hadoopNS:8020/flink/checkpoints/${job_name}/chk-7/_metadata
