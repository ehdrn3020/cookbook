
## Cross Cluster Data Mirroring 
- MM2 전용 커넥터가 Kafka 배포판에 이미 들어 있고, connect-mirror-maker.sh가 알아서 띄웁니다. 
- 별도 Connect 클러스터나 플러그인 다운로드 없이 properties 파일 하나로 실행됩니다.
```aiignore
참조 : https://kafka.apache.org/41/operations/geo-replication-cross-cluster-data-mirroring/
```

### What Are Replication Flows
- 토픽(레코드), 토픽 설정, 컨슈머 그룹과 그 오프셋, ACL까지 복제
- 흐름은 단방향이다
- 흐름마다 독립적으로 설정


### MM2 Configuration File Syntax
- connect-mirror-maker.properties 파일로 설정
- 커넥터는 쓰이지만, Connect 설정을 그냥 갖다 쓰면 된다
  - MM2가 Connect 기반이라, Connect·source 커넥터·sink 커넥터 설정을 이름을 바꾸거나 접두사를 붙일 필요 없이 그대로 씁니다. Connect 문서에 나오는 키를 mm2.properties에 복사해 넣으면 동작합니다.
```aiignore
clusters = primary, secondary              # 별칭 2개 선언
primary.bootstrap.servers = ...            # 각 별칭의 접속 정보
secondary.bootstrap.servers = ...

primary->secondary.enabled = true          # 이 흐름 켜기 (기본 false)
primary->secondary.topics = foobar-topic, quux-.*   # 이 토픽들만
```
- 예외는 tasks.max 하나
  - 여러 MirrorMaker 프로세스에 작업을 고르게 나누려면 최소 2, 가급적 더 높게 잡으라고 권장합니다. 기준은 가용 하드웨어 자원과 복제할 토픽-파티션 총 개수입니다.
- 클러스터별로 다르게 주려면 {cluster}.{설정}
  - 접두사 없이 쓰면 전역, 특정 클러스터에만 적용하려면 별칭을 앞에 붙입니다.
```aiignore
tasks.max = 5                          # 전역
B.offset.storage.topic = my-mm2-offsets # B에만
```


### Exactly once
- 버전 3.5.0부터 전용 MirrorMaker 클러스터에서 정확히 한 번 쓰기(exactly-once semantics)가 지원됩니다. 
- 단 설정 한 줄로 끝나는 게 아니라 REST 통신 활성화까지 세트로 켜야 합니다.

1) 타깃 클러스터에 exactly-once 켜기
```aiignore
B.exactly.once.source.support = enabled
```
2) 노드 간 REST 통신 열기 (필수)
```aiignore
dedicated.mode.enable.internal.rest = true
listeners = http://localhost:8080
```
3) 소스 consumer를 read_committed로
```aiignore
A.consumer.isolation.level = read_committed
```


### Creating and Enabling Replication Flows
- 복제 흐름 만들기는 2단계이다.
- 1단계 : 클러스터 정의 (필수 2개)
```aiignore
clusters = A, B
A.bootstrap.servers = a-broker1:9092,a-broker2:9092
B.bootstrap.servers = b-broker1:9092,b-broker2:9092
```
- 2단계 : 흐름 켜기 (명시적)
```aiignore
A->B.enabled = true
```
- 최소 설정
```aiignore
clusters = A, B
A.bootstrap.servers = ...
B.bootstrap.servers = ...

A->B.enabled = true
A->B.topics = cdc\..*                    # ①번 기본값 좁히기, 없으면 다가져옴
replication.policy.class = org.apache.kafka.connect.mirror.IdentityReplicationPolicy   # ②번 prefix 제거
```
### Preventing Configuration Conflicts
- 같은 타깃 클러스터를 향하는 MirrorMaker 프로세스들은 설정을 공유한다. 그래서 설정이 서로 다르면 리더로 뽑힌 한 대의 설정만 살아남고 나머지는 무시된다.
- 왜 이런 일이 생기나 
  - MM2는 Connect 기반이라 프로세스들이 타깃 클러스터(B)의 config 토픽을 통해 설정을 공유합니다. 같은 B를 바라보는 프로세스들은 자동으로 한 덩어리의 분산 클러스터가 됩니다. 
  - 여기서 "합쳐진다"가 아니라 **"하나가 이긴다"**가 포인트입니다.
- 해결 방법
  - 같은 타깃 클러스터를 바라보는 MM2 프로세스들은 설정을 공유하므로, 설정이 서로 다르면 리더로 뽑힌 한 대의 설정만 살아남고 나머지는 무시됩니다. 
  - 따라서, 같은 타깃 클러스터를 바라보는 모든 MM2 프로세스들은 동일한 설정을 사용해야 합니다.


### MM2를 어느 클러스터 쪽에 둘 것인가
- 결론 : **화살표가 받는 쪽, 즉 타깃 클러스터 가까이 둔다.**
- 원칙은 **consume from remote, produce to local** (원격에서 읽고 로컬에 쓴다)
```aiignore
  remote-kafka01                              local-storage01
  (A : 원격, 소스)                             (B : 로컬, 타깃)
        │                                            │
        │                                   ┌────────┴────────┐
        └────────── consume ────────────────┤      MM2        │
                   (원격 read)               │  여기에 배치     │
                                            └────────┬────────┘
                                                     │ produce
                                                     ▼ (로컬 write)
                                              local-storage01
```
#### 왜 타깃 쪽인가
- Kafka **producer가 consumer보다 네트워크에 더 취약**하다.
- consumer는 끊겨도 offset을 들고 있다가 재연결해서 이어 읽으면 그만이다. 반면 producer는 원격 구간에서 타임아웃·재시도가 걸리면 처리량이 떨어지고, 설정에 따라 유실이나 중복까지 번진다.
- 공식 문서도 이 배치를 best practice로 명시한다. (참조 : 위 geo-replication 문서)
- Preventing Configuration Conflicts와 같이 보면 이해가 쉽다. MM2는 **타깃 클러스터의 config 토픽으로 설정을 공유**하므로, 태생적으로 타깃 쪽에 붙는 구조다.


### 실전 설정 예제
- 원격 클러스터의 MSSQL CDC 토픽을 로컬로 단방향 복제하는 구성
```aiignore
# ── 1. 클러스터 별칭 선언 (필수, 2개) ─────────────────────────────
clusters = remote, local

# 별칭별 접속 정보. MM2는 local 쪽에 띄운다 (consume from remote, produce to local)
remote.bootstrap.servers = remote-broker1:9092,remote-broker2:9092,remote-broker3:9092
local.bootstrap.servers  = local-broker1:9092,local-broker2:9092,local-broker3:9092

# ── 2. 흐름 방향 ────────────────────────────────────────────────
# 흐름은 단방향. 기본값이 false지만 역방향을 명시해 의도를 남긴다
remote->local.enabled = true
local->remote.enabled = false

# ── 3. 복제 대상 토픽 ───────────────────────────────────────────
# 정규식이므로 점(.)은 반드시 이스케이프 — 안 하면 임의 문자가 되어 의도보다 넓게 잡힌다
# Debezium MSSQL 토픽 구조 : {topic.prefix}.{database}.{schema}.{table}
remote->local.topics = stg_mssql_orderdb01\.ORDERDB\.dbo\..*

# ── 4. 토픽명 정책 ─────────────────────────────────────────────
# 기본 DefaultReplicationPolicy는 타깃 토픽에 "remote." prefix를 붙인다.
# IdentityReplicationPolicy는 원본 토픽명을 그대로 유지 (소스와 이름이 같아짐)
# ※ 단방향에서만 안전. 양방향이면 A→B→A 무한 복제 루프를 못 막는다
replication.policy.class = org.apache.kafka.connect.mirror.IdentityReplicationPolicy

# ── 5. Replication Factor ──────────────────────────────────────
# 아래는 브로커 1대 기준 값. 운영은 전부 3 권장 (RF=1은 브로커 장애 시 그대로 유실)
replication.factor = 1                          # 복제해서 만드는 데이터 토픽의 RF

# MM2가 쓰는 내부 토픽 3종
checkpoints.topic.replication.factor = 1        # 컨슈머 그룹 오프셋 체크포인트
heartbeats.topic.replication.factor = 1         # 흐름 생존 확인용 하트비트
offset-syncs.topic.replication.factor = 1       # 소스↔타깃 오프셋 매핑 테이블

# Connect 런타임이 쓰는 내부 토픽 3종 (Default Topic 문서 참조)
offset.storage.replication.factor = 1           # connect-offsets
status.storage.replication.factor = 1           # connect-status
config.storage.replication.factor = 1           # connect-configs

# ── 6. 병렬도 ──────────────────────────────────────────────────
# 최소 2 권장. 기준은 하드웨어 자원과 복제할 토픽-파티션 총 개수
tasks.max = 2

# ── 7. 부가 기능 ───────────────────────────────────────────────
# 컨슈머 그룹 오프셋을 타깃에 동기화할지. 타깃에서 이어 읽을 컨슈머가 없으면 false
remote->local.sync.group.offsets.enabled = false

# 소스 토픽의 설정(retention.ms, cleanup.policy 등)을 타깃 토픽에 계속 맞출지. 기본값 true.
remote->local.sync.topic.configs.enabled = false

# 하트비트 발행. 복제 흐름이 살아있는지 모니터링하려면 true
remote->local.emit.heartbeats.enabled = true
```

### 실행
```aiignore
export JAVA_HOME=/usr/lib/jvm/jdk-1.8.0-openjdk
export KAFKA_LOG4J_OPTS="-Dlog4j.configuration=file:/kafka/default/config/mm2-log4j.properties"

nohup /kafka/default/bin/connect-mirror-maker.sh \
/kafka/default/config/connect-mirror-maker-source_cluster.properties \
--clusters target_cluster > /dev/null 2>&1 &

# 로그 경로
tail -f /kafka/default/logs/mm2.log
```