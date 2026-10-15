# MirrorMaker 2 설정 예제

> 작성일: 2026-10-14

- 참조 : [Geo-Replication (Cross-Cluster Data Mirroring)](https://kafka.apache.org/41/operations/geo-replication-cross-cluster-data-mirroring/)
- 참조 : [MirrorMaker 2 개념 정리](mirror-maker.md) — 복제 흐름, 배치 위치, exactly-once 등 개념은 이쪽
- CDC 원본 클러스터의 Debezium 토픽을 분석용 클러스터로 **단방향 복제**하는 실제 properties 한 벌을 옵션 단위로 읽는다.

## 전체 설정

```properties
# connect-mirror-maker-targethost.properties

clusters = targethost, endhost

targethost.bootstrap.servers = targethost1:9092,targethost2:9092,targethost3:9092,targethost4:9092,targethost5:9092
endhost.bootstrap.servers    = endhost1:9092,endhost2:9092,endhost3:9092

# consumer
targethost.consumer.auto.offset.reset = latest
targethost.consumer.fetch.max.bytes = 10485760
targethost.consumer.max.partition.fetch.bytes = 1048576

# producer
endhost.producer.buffer.memory = 33554432
endhost.producer.compression.type = lz4
endhost.producer.linger.ms = 20
endhost.producer.max.block.ms = 120000

# 단방향만
targethost->endhost.enabled = true
endhost->targethost.enabled = false

# 대상 토픽
targethost->endhost.topics = prod_mariadb_shopdb\.shop_db\.(order_item_tbl|order_info_tbl), prod_mssql_billdb01\.BILLDB\.dbo\.(TPayMst|TPayUser|TCouponUser|TItemHoldMst)

# 토픽명 prefix 제거
replication.policy.class = org.apache.kafka.connect.mirror.IdentityReplicationPolicy

replication.factor = 3
checkpoints.topic.replication.factor = 3
heartbeats.topic.replication.factor = 3
offset-syncs.topic.replication.factor = 3
offset.storage.replication.factor = 3
status.storage.replication.factor = 3
config.storage.replication.factor = 3

tasks.max = 1

targethost->endhost.sync.group.offsets.enabled = false
targethost->endhost.emit.heartbeats.enabled = true
targethost->endhost.sync.topic.configs.enabled = false

# 운영 클러스터에 내부 토픽을 만들지 않도록
targethost->endhost.offset-syncs.topic.location = target
```

## 클러스터 별칭

```properties
clusters = targethost, endhost

targethost.bootstrap.servers = targethost1:9092,...,targethost5:9092
endhost.bootstrap.servers    = endhost1:9092,endhost2:9092,endhost3:9092
```

- `clusters`에 선언한 이름이 이후 모든 설정의 **접두사**가 된다. `{별칭}.{설정}`은 그 클러스터에만, 접두사 없이 쓰면 전역이다.
- 별칭은 임의로 정하는 라벨이지만, 한 번 정하면 `offset-syncs`·`checkpoints` 내부 토픽 이름과 컨슈머 그룹 ID에 박히므로 **운영 중 변경하면 복제가 처음부터 다시 시작**된다.
- MM2 프로세스 자체는 **받는 쪽(`endhost`)** 에 띄운다. consume from remote, produce to local.
- **이름 주의** : 여기서 `targethost`는 읽어오는 **소스**, `endhost`가 써 넣는 **타깃**이다. MM2 용어의 `target`(= 타깃 클러스터)과 철자가 겹치니, 아래 `offset-syncs.topic.location = target`의 `target`은 `targethost`가 아니라 **`endhost`를 가리킨다**는 점을 기억한다.

## 튜닝 지점 한눈에 보기

```
┌─ targethost  (소스, 브로커 5대) ────────────────────────────┐
│ targethost1:9092 ... targethost5:9092                       │
│ ※ MM2는 여기에 띄우지 않는다                                │
└─────────────────────────────────────────────────────────────┘
                               │  consume  (원격 읽기 — 끊겨도 offset 들고 재연결)
                               ▼
              ┌──────────────────────────────────────────────────┐
              │ auto.offset.reset         = latest               │
              │ fetch.max.bytes           = 10MB    (기본 50MB)  │
              │ max.partition.fetch.bytes = 1MB     (기본값)     │
              │                                                  │
              │ → 받는 덩어리를 작게. 받는 즉시 흘려보내므로     │
              │   consumer 쪽에 쌓아둘 이유가 없다               │
              └──────────────────────────────────────────────────┘
                               │
                               ▼
┌─ MM2 프로세스  (endhost 쪽에 배치) ─────────────────────────┐
│ tasks.max = 1        ← consumer·producer 한 쌍              │
│                                                             │
│ -Xmx 1G              힙                                     │
│ MaxMetaspaceSize  256m   ┐ 힙 밖                            │
│ MaxDirectMemorySize 256m ┘ (소켓 버퍼)                      │
│                                                             │
│ 총 점유 ≈ 1.6~1.8G — cgroup 한도는 여기에 맞춘다            │
└─────────────────────────────────────────────────────────────┘
                               │
                               ▼
              ┌────────────────────────────────────────────────────────┐
              │ linger.ms        = 20ms   배치를 모은다                │
              │ compression.type = lz4    모인 배치를 압축  ◀ 효과 최대│
              │ buffer.memory    = 32MB   전송 대기 버퍼               │
              │ max.block.ms     = 120s   버퍼 full 시 대기 한도       │
              │                                                        │
              │ → 위 두 줄은 세트. 배치가 커야 압축이 먹는다           │
              │ → 아래 두 줄도 세트. 버퍼가 차면 send()가 막힌다       │
              └────────────────────────────────────────────────────────┘
                               │  produce  (로컬 쓰기 — 네트워크에 취약한 쪽)
                               ▼
┌─ endhost  (타깃, 브로커 3대) ───────────────────────────────┐
│ endhost1:9092  endhost2:9092  endhost3:9092                 │
│ replication.factor = 3   ← 브로커 수가 상한                 │
└─────────────────────────────────────────────────────────────┘
```

- **위에서 아래로 한 번만 흐른다.** 소스에서 읽어(consume) MM2 힙을 거쳐 타깃에 쓴다(produce). 가운데 두 개의 작은 박스가 각각 consumer / producer 튜닝 지점이다.
- consumer 쪽은 **작게**, producer 쪽은 **모아서 압축**이 이 설정의 성격이다. MM2는 받은 즉시 다시 보내므로 중간에 쌓아둘 이유가 없고, 병목은 쓰는 쪽 네트워크에 있기 때문이다.
- 가운데 JVM 박스가 **위아래 모든 버퍼를 담는 그릇**이다. `tasks.max`를 올리면 consumer·producer 한 쌍이 통째로 늘어나 버퍼도 그만큼 곱해진다.

## 소스 consumer 튜닝

```properties
targethost.consumer.auto.offset.reset = latest
targethost.consumer.fetch.max.bytes = 10485760           # 10MB  (기본 50MB)
targethost.consumer.max.partition.fetch.bytes = 1048576  # 1MB   (기본값 그대로)
```

| 옵션 | 기본값 | 역할 |
| --- | --- | --- |
| `auto.offset.reset` | `latest` | 복제 시작 시점에 **과거 데이터를 안 가져온다.** 지금부터 들어오는 것만 복제 |
| `fetch.max.bytes` | 50MB | 한 번의 fetch 응답으로 브로커에서 받아올 **총량** 상한 |
| `max.partition.fetch.bytes` | 1MB | 그중 **파티션 하나당** 상한 |

- `auto.offset.reset`은 **MM2 기본값이 `latest`**다. Kafka consumer 일반 기본값인 `earliest`와 다르니 헷갈리기 쉽다. 명시해두면 의도가 분명해진다.
  - `earliest`로 두면 MM2를 처음 띄우는 순간 소스 토픽의 **보존 기간 전체를 끌어온다.** CDC 토픽이 며칠치 쌓여 있으면 그대로 폭주한다.
  - 반대로 과거 데이터까지 옮기는 게 목적이라면 `earliest`여야 한다. 한 번 복제가 시작되면 offset이 기록되므로 나중에 바꿔도 소급되지 않는다.
- `fetch.max.bytes = 10MB`는 기본값 50MB를 **줄인** 값이다. 한 번에 받는 덩어리를 작게 해 **힙 사용량과 응답 지연의 튐을 억제**하는 쪽 선택이다. 복제 처리량만 보면 키우는 게 유리하지만, MM2는 받은 즉시 타깃으로 다시 produce하므로 consumer 쪽에 크게 쌓아둘 이유가 없다.
- `max.partition.fetch.bytes = 1MB`는 **기본값 그대로**다. 적어둔 이유는 위 `fetch.max.bytes`와의 비율을 눈에 보이게 하기 위함이다. 10MB ÷ 1MB이므로 파티션 10개분을 한 번에 받는 셈이고, 파티션 하나가 응답을 독식하지 못한다.
  - 단, 이 값보다 **큰 단일 메시지**가 있으면 그 파티션은 진행이 막힌다. 브로커의 `max.message.bytes`보다 작게 잡으면 안 된다.

## 타깃 producer 튜닝

```properties
endhost.producer.buffer.memory = 33554432   # 32MB   (기본값 그대로)
endhost.producer.compression.type = lz4     #        (기본 none)
endhost.producer.linger.ms = 20             # 20ms   (기본 0)
endhost.producer.max.block.ms = 120000      # 120초  (기본 60초)
```

### compression.type = lz4

- **이 네 줄 중 효과가 가장 큰 설정이다.** 원격 복제는 네트워크 구간이 병목이므로, 보내기 전에 배치를 압축해 전송량을 줄인다.
- CDC 레코드는 JSON이고 같은 스키마가 반복되므로 **압축률이 특히 잘 나온다.**
- `lz4`는 압축률보다 **속도**를 택한 선택이다. CPU를 덜 쓰면서 충분한 효과를 낸다. 더 줄이고 싶으면 `zstd`, 호환성이 걸리면 `snappy`.
- 압축은 **배치 단위**로 일어난다. 그래서 아래 `linger.ms`와 묶여야 의미가 산다.

### linger.ms = 20

- producer가 배치를 채우려고 **최대 20ms 기다렸다가** 보낸다. 기본값 0은 "모이는 대로 즉시 전송"이다.
- 기다리는 만큼 배치가 커지고, 배치가 커지면 **압축률이 올라가고 요청 횟수가 준다.** `compression.type`과 세트로 보는 이유다.
- 비용은 **20ms의 추가 지연**이다. 복제는 원래 비동기이므로 이 정도는 보통 감수할 만하다. 실시간성이 중요한 흐름이면 낮춘다.

### buffer.memory = 32MB

- producer가 전송 전 레코드를 모아두는 전체 버퍼. **기본값 그대로**다.
- 버퍼가 차면 `send()`가 **블로킹**된다. 타깃이 느릴 때 메모리를 무한히 먹는 대신 상류에 역압(backpressure)이 걸리는 구조다.

### max.block.ms = 120000

- 그 **블로킹을 얼마나 버틸지**다. 기본 60초에서 2배로 늘렸다. 이 시간을 넘기면 `TimeoutException`이 나고 태스크가 실패한다.
- `buffer.memory`와 짝이다. 버퍼가 가득 찬 상태 — 타깃 브로커가 느리거나 일시적으로 흔들릴 때 — 에서 **바로 죽지 않고 더 기다려주게** 하는 설정이다.
- 메타데이터 조회(`send()`가 처음 보는 토픽의 파티션 정보를 가져올 때)에도 같은 상한이 적용된다.
- 늘린다고 문제가 해결되는 건 아니다. 120초를 꽉 채우는 일이 반복되면 **타깃 쓰기 용량이 모자란 것**이므로 `tasks.max`나 타깃 브로커 쪽을 봐야 한다.

## 흐름 방향

```properties
targethost->endhost.enabled = true
endhost->targethost.enabled = false
```

- `enabled`의 기본값은 `false`다. 역방향을 굳이 `false`로 적는 건 **"양방향이 아니다"라는 의도를 설정 파일에 남기기 위한 것**이다. 아래 `IdentityReplicationPolicy`와 짝이 되는 전제라 명시해두는 편이 안전하다.

## 대상 토픽

```properties
targethost->endhost.topics = prod_mariadb_shopdb\.shop_db\.(order_item_tbl|order_info_tbl), prod_mssql_billdb01\.BILLDB\.dbo\.(TPayMst|TPayUser|TCouponUser|TItemHoldMst)
```

- **정규식**이다. 쉼표로 여러 패턴을 나열하고, `(a|b|c)`로 같은 prefix의 테이블을 묶는다.
- 점(`.`)은 반드시 `\.`로 이스케이프한다. 안 하면 임의 문자가 되어 의도보다 넓게 잡힌다.
- 토픽 구조는 Debezium 규칙을 그대로 따른다.
  - MariaDB : `{topic.prefix}.{database}.{table}`
  - MSSQL : `{topic.prefix}.{database}.{schema}.{table}`
- 지정하지 않으면 기본값이 `.*`라 **소스의 모든 토픽을 복제한다.** 반드시 좁혀야 한다.
- 소스에 새 토픽이 생기면 주기적 스캔(`refresh.topics.interval.seconds`, 기본 600초)으로 자동 추가된다. 정규식에 걸리는 토픽이 늘면 복제 대상도 조용히 늘어난다는 뜻이다.

## 토픽명 정책

```properties
replication.policy.class = org.apache.kafka.connect.mirror.IdentityReplicationPolicy
```

- 기본 `DefaultReplicationPolicy`는 복제해 만든 토픽에 `targethost.` 라는 **소스 별칭 prefix를 붙인다.** 다운스트림 컨슈머가 원본과 같은 토픽명을 기대하면 그대로 깨진다.
- `IdentityReplicationPolicy`는 원본 토픽명을 유지한다.
- **단방향에서만 안전하다.** 이름이 같아지므로 양방향이면 A→B→A 무한 복제 루프를 구분할 방법이 없다. 위에서 역방향을 `false`로 못박은 이유다.
- 전역 설정이라 흐름별 접두사를 붙이지 않는다.

## Replication Factor

```properties
replication.factor = 3                          # 복제해서 만드는 데이터 토픽

checkpoints.topic.replication.factor = 3        # 컨슈머 그룹 오프셋 체크포인트
heartbeats.topic.replication.factor = 3         # 흐름 생존 확인용 하트비트
offset-syncs.topic.replication.factor = 3       # 소스↔타깃 오프셋 매핑

offset.storage.replication.factor = 3           # connect-offsets
status.storage.replication.factor = 3           # connect-status
config.storage.replication.factor = 3           # connect-configs
```

- 세 묶음의 성격이 다르다.
  - `replication.factor` — MM2가 타깃에 **새로 만드는 데이터 토픽**의 RF
  - `checkpoints` / `heartbeats` / `offset-syncs` — **MM2 전용** 내부 토픽
  - `offset.storage` / `status.storage` / `config.storage` — **Connect 런타임** 내부 토픽 ([Default Topics](default-topics.md) 참조)
- 이 값은 **토픽이 처음 생성될 때만 적용된다.** 이미 만들어진 토픽의 RF는 바뀌지 않으므로, 나중에 올리려면 `kafka-reassign-partitions.sh`를 따로 돌려야 한다.
- 브로커 수보다 큰 RF를 주면 토픽 생성 자체가 실패한다. 타깃이 3대이므로 3이 상한이다.

## 병렬도

```properties
tasks.max = 1
```

- 복제 작업을 몇 개의 태스크로 쪼갤지. 공식 문서는 **최소 2, 가급적 더 높게**를 권장한다.
- 여기서 1인 건 대상 토픽이 적고 트래픽이 감당 가능한 수준이기 때문으로 보인다. 다만 **태스크 1개는 복제 처리량의 상한도 1개 스레드로 묶인다**는 뜻이라, 토픽-파티션이 늘면 가장 먼저 병목이 되는 값이다.
- 올려도 **토픽-파티션 총 개수를 넘으면 남는 태스크는 논다.**
- MM2 노드를 2대 이상 올려 HA를 구성한다면 이 값이 **1이면 한 대가 놀게 된다.** [다중 노드 HA 구성](mirror-maker-2-multi-node-high-availability.md) 참조.

## 부가 기능

```properties
targethost->endhost.sync.group.offsets.enabled = false
targethost->endhost.emit.heartbeats.enabled = true
targethost->endhost.sync.topic.configs.enabled = false
```

### sync.group.offsets.enabled

- 소스의 컨슈머 그룹 오프셋을 타깃 `__consumer_offsets`에 반영할지. 기본값 `false`.
- **재해 복구(DR) 목적**이 아니면 켤 이유가 없다. 타깃에서 같은 그룹 ID로 이어 읽을 컨슈머가 있을 때만 의미가 있다.

### emit.heartbeats.enabled

- 소스 클러스터의 `heartbeats` 토픽에 주기적으로 레코드를 발행한다. 기본값 `true`.
- 복제 흐름이 살아있는지 확인하는 용도다. 이 토픽이 멈추면 MM2가 죽었거나 소스 연결이 끊긴 것이다.

### sync.topic.configs.enabled

- 소스 토픽의 **설정**(`retention.ms`, `cleanup.policy`, `min.insync.replicas` 등)을 타깃 토픽에 계속 맞출지. 기본값 `true`이고, `sync.topic.configs.interval.seconds`(기본 600초) 주기로 재동기화한다.
- 복제 대상은 데이터가 아니라 **토픽 속성**이다. RF와 파티션 수는 이 옵션이 관여하지 않는다.
- `false`로 두는 경우
  - 타깃에서 **보존 기간을 다르게** 가져갈 때. 켜두면 바꿔놔도 10분 뒤 소스 값으로 되돌아간다.
  - 소스의 `min.insync.replicas`가 타깃 브로커 수보다 클 때. 그대로 따라붙으면 **타깃에 produce가 막힌다.**
  - 타깃 토픽을 IaC나 별도 정책으로 관리할 때. 설정 주체가 둘이 되면 충돌한다.

## offset-syncs 토픽 위치

```properties
targethost->endhost.offset-syncs.topic.location = target
```

- **이 설정 한 줄의 의미가 크다.** `offset-syncs`는 소스 offset ↔ 타깃 offset 매핑을 저장하는 MM2 내부 토픽인데, **기본값이 `source`다.** 즉 기본 동작은 **소스(운영) 클러스터에 MM2가 내부 토픽을 만들고 쓴다.**
- 운영 클러스터를 건드리지 않아야 하는 상황 — 토픽 생성 권한이 없거나, 운영 측에서 외부 프로세스의 쓰기를 금지하는 경우 — 이면 `target`으로 돌려야 한다.
- 소스에 대해 **읽기 권한만으로 복제가 가능**해진다는 것이 실질적인 이득이다.
- 다만 `emit.heartbeats.enabled = true`는 여전히 **소스 클러스터의 heartbeats 토픽에 쓴다.** 소스를 완전히 읽기 전용으로 두려면 이쪽도 `false`로 꺼야 한다. 위 설정은 그 둘이 어긋나 있는 상태다.

## 실행

```bash
export JAVA_HOME=/usr/lib/jvm/jdk-1.8.0-openjdk
export KAFKA_LOG4J_OPTS="-Dlog4j.configuration=file:/kafka/default/config/mm2-log4j.properties"

# 리소스 제한
export KAFKA_HEAP_OPTS="-Xms512M -Xmx1G"
export KAFKA_OPTS="-XX:MaxMetaspaceSize=256m -XX:MaxDirectMemorySize=256m"

nohup /kafka/default/bin/connect-mirror-maker.sh \
  /kafka/default/config/connect-mirror-maker-targethost.properties \
  --clusters endhost > /dev/null 2>&1 &

tail -f /kafka/default/logs/mm2.log
```

- `--clusters endhost` : **이 프로세스가 어느 클러스터를 담당할지** 지정한다. 안 주면 properties에 선언된 모든 클러스터를 한 프로세스가 맡는다.
- 받는 쪽 별칭(`endhost`)을 주는 것이 "타깃 쪽에 배치한다"는 원칙의 실제 구현이다.
- 환경변수는 **스크립트 실행 전에 export** 해야 한다. `connect-mirror-maker.sh`가 내부의 `kafka-run-class.sh`를 통해 읽어가는 값이라, 나중에 설정하면 반영되지 않는다.

## 리소스 제한

```bash
export KAFKA_HEAP_OPTS="-Xms512M -Xmx1G"
export KAFKA_OPTS="-XX:MaxMetaspaceSize=256m -XX:MaxDirectMemorySize=256m"
```

| 옵션 | 영역 | 역할 |
| --- | --- | --- |
| `-Xms` / `-Xmx` | 힙 | JVM 힙 초기 / 최대 크기 |
| `MaxMetaspaceSize` | 힙 밖 | 클래스 메타데이터 영역 상한 |
| `MaxDirectMemorySize` | 힙 밖 | NIO 다이렉트 버퍼 상한 |

- 두 변수는 역할이 다르다. `KAFKA_HEAP_OPTS`는 **힙만**, `KAFKA_OPTS`는 그 외 JVM 인자를 넘긴다. `KAFKA_OPTS`에 `-Xmx`를 같이 넣으면 중복 지정이 되니 섞지 않는다.
- MM2를 `connect-mirror-maker.sh`로 띄우면 기본 힙이 **GB 단위로 넉넉하게** 잡힌다. 위처럼 명시하는 건 **한 서버에 다른 프로세스와 같이 올릴 때 상한을 못박기 위함**이다.

### 왜 힙 밖까지 막아야 하나

- **힙만 제한하면 프로세스 전체 메모리는 제한되지 않는다.** 실제 점유는 대략 이렇게 쌓인다.

```
힙 1G  +  Metaspace 256m  +  Direct 256m  +  스레드 스택  +  JVM 자체 오버헤드
≈ 1.6~1.8G
```

- 컨테이너나 cgroup 메모리 한도를 잡을 때 **힙 크기만 보고 1G로 주면 OOM Kill**을 당한다. 위 합계에 여유를 더해 잡아야 한다.
- `MaxDirectMemorySize`가 특히 중요하다. Kafka 클라이언트는 **소켓 읽기/쓰기에 다이렉트 버퍼를 쓴다.** 기본값은 사실상 힙 최대치와 같아서, 막아두지 않으면 힙은 1G인데 힙 밖에서 1G를 더 먹는 상황이 생긴다.

### 위쪽 튜닝값과의 관계

- 힙 1G는 **아래 두 값을 담고도 남아야 한다.**
  - `endhost.producer.buffer.memory = 32MB` — producer 전송 버퍼
  - `targethost.consumer.fetch.max.bytes = 10MB` — fetch 응답 1회분
- `tasks.max`를 올리면 **태스크마다 consumer와 producer가 따로 생긴다.** 즉 위 값들이 태스크 수만큼 곱해지므로, 병렬도를 올릴 때는 힙도 같이 올려야 한다. 현재 `tasks.max = 1`이라 1G로 버티는 구성이다.
- `compression.type = lz4`도 압축 버퍼를 쓴다. 양이 크진 않지만 힙을 빠듯하게 잡을 때는 염두에 둔다.

### 증상별로 보는 곳

| 증상 | 의심할 설정 |
| --- | --- |
| `OutOfMemoryError: Java heap space` | `-Xmx`, `buffer.memory`, `fetch.max.bytes`, `tasks.max` |
| `OutOfMemoryError: Direct buffer memory` | `MaxDirectMemorySize` |
| `OutOfMemoryError: Metaspace` | `MaxMetaspaceSize` |
| 로그 없이 프로세스가 사라짐 | 힙이 아니라 **OS/컨테이너의 OOM Kill** — `dmesg` 확인 |

- 마지막 줄이 헷갈리기 쉽다. JVM이 던진 OOM은 로그가 남지만, **커널이 죽인 경우는 애플리케이션 로그에 아무것도 안 남는다.**

## 관련 문서

- [MirrorMaker 2 다중 노드 HA 구성](mirror-maker-2-multi-node-high-availability.md) — 같은 properties로 2대를 올렸을 때의 장애 조치
- [MirrorMaker 2](mirror-maker.md) — 복제 흐름, MM2 배치 위치, exactly-once
- [Kafka Connect 기본 토픽](default-topics.md) — connect-offsets / status / configs
- [__consumer_offsets 토픽](../kafka/consumer-offsets-topic.md)
