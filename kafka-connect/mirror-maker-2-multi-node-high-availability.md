# MirrorMaker 2 다중 노드 HA 구성

> 작성일: 2026-10-14

- 참조 : [Geo-Replication (Cross-Cluster Data Mirroring)](https://kafka.apache.org/41/operations/geo-replication-cross-cluster-data-mirroring/)
- 참조 : [MirrorMaker 2 설정 예제](mirror-maker-2-configuration-example.md) — 아래에서 다루는 properties 한 벌
- 참조 : [MirrorMaker 2 개념 정리](mirror-maker.md)

**같은 properties로 2대를 올리면 한 대가 죽어도 다른 한 대가 이어받는다.** 단 전제 조건이 두 개 있고, **즉시가 아니라 기본 설정이면 최대 5분**이 걸린다.

## 왜 이어받을 수 있나 — 상태가 로컬에 없다

```
┌────────────────────────────────────────────────────────────────┐
│ MM2 노드 (둘 다 endhost 쪽에 배치)                             │
│                                                                │
│   mm2node1  ┐                                                  │
│             ├─ 하나의 Connect 워커 그룹                        │
│   mm2node2  ┘   (endhost 의 그룹 코디네이터가 관리)            │
│                                                                │
│   ※ 두 노드 모두 로컬에 진행 상태를 두지 않는다                │
└────────────────────────────────────────────────────────────────┘
                              │
                        진행 위치를 커밋
                              ▼
┌────────────────────────────────────────────────────────────────┐
│ endhost (타깃 클러스터)의 내부 토픽                            │
│                                                                │
│   mm2-offsets.targethost.internal   어디까지 복제했나          │
│   mm2-status.targethost.internal    커넥터·태스크 상태         │
│   mm2-configs.targethost.internal   커넥터 설정                │
│                                                                │
│   → 상태가 여기 있으므로 어느 노드가 이어받아도 된다           │
└────────────────────────────────────────────────────────────────┘
```

- 두 노드는 각자 도는 독립 프로세스가 아니라, **타깃 클러스터의 그룹 코디네이터가 관리하는 하나의 Connect 워커 그룹**을 이룬다.
- 복제 진행 위치는 노드의 로컬 디스크가 아니라 **타깃 클러스터의 내부 토픽**에 커밋된다. 내부 토픽 이름에 소스 별칭(`targethost`)이 들어가는 것이 dedicated mode의 명명 규칙이다.
- `mm2node1`이 죽으면 그룹에서 빠지고, 리밸런스를 거쳐 `mm2node2`가 그 태스크를 받아 **마지막 커밋 지점부터** 이어간다. 로컬 상태가 없어서 가능한 구조다.
- 이어받는 지점이 "마지막 커밋"이므로 **중복은 생길 수 있다.** at-least-once이지 exactly-once가 아니다. 정확히 한 번이 필요하면 [개념 문서](mirror-maker.md)의 exactly-once 절을 본다.

## 전제 조건 ① — 내부 REST 통신

```properties
dedicated.mode.enable.internal.rest = true
listeners = http://mm2node1:8080      # 노드마다 자기 호스트명으로
```

- **노드마다 값이 다른 유일한 줄이다.** 나머지는 전부 같아야 한다(아래 ②).
- dedicated mode(`connect-mirror-maker.sh`)에서 **워커끼리 직접 통신하는 통로**다. 리밸런스 때 커넥터 설정을 주고받는 데 쓴다.
- 이게 꺼져 있으면 노드를 2대 올려도 **서로를 모른 채 각자 돈다.** 협조가 안 되므로 HA가 아니라 중복 실행에 가까워진다.
- 노드 간 해당 포트를 **방화벽에서 열어야 한다.** 흔히 빠뜨리는 지점이다.

## 전제 조건 ② — properties가 완전히 동일할 것

- 같은 타깃 클러스터를 바라보는 MM2 프로세스들은 **config 토픽을 통해 설정을 공유**한다. 설정이 서로 다르면 합쳐지는 게 아니라 **리더로 뽑힌 한 대의 설정만 살아남고 나머지는 무시된다.**
- 그래서 설정이 어긋나면 **증상이 조용하다.** 에러 없이 의도와 다른 설정으로 돌기 때문에, 토픽 목록이나 RF가 한쪽 파일 기준으로만 반영된 걸 나중에 발견하게 된다.
- `listeners`만 호스트별로 다르고, 나머지는 파일을 그대로 복사해 쓴다.

```bash
# 배포 후 두 노드의 파일이 같은지 확인
md5sum /kafka/default/config/connect-mirror-maker-targethost.properties
```

## 즉시가 아닌 이유 — scheduled.rebalance.max.delay.ms

```properties
scheduled.rebalance.max.delay.ms = 300000    # 기본값 5분
```

- Connect는 **점진적 협력 리밸런싱**(incremental cooperative rebalancing)을 쓴다. 워커가 사라져도 **바로 태스크를 재배치하지 않고 돌아오기를 기다린다.**
- 목적은 **불필요한 리밸런스를 막는 것**이다. 롤링 재시작이나 짧은 GC 정지처럼 금방 복귀하는 경우까지 매번 전체 재배치를 하면 그 자체가 더 큰 중단이 된다.
- 그래서 `mm2node1`이 죽은 뒤 **최대 5분 동안은 그 태스크가 멈춰 있다.** 복제 지연이 그만큼 생긴다.
- 더 빨리 넘기고 싶으면 낮춘다. 트레이드오프가 분명하다.

| 값 | 장애 조치 | 대가 |
| --- | --- | --- |
| 300000 (기본) | 최대 5분 지연 | 롤링 재시작 시 리밸런스 없음 |
| 60000 (1분) | 최대 1분 지연 | 재시작·일시 정지마다 리밸런스 발생 |
| 0 | 즉시 | 워커가 잠깐만 흔들려도 전체 재배치 |

- **0은 권장하지 않는다.** 네트워크가 잠깐 끊길 때마다 태스크가 이리저리 옮겨 다니며 오히려 복제가 불안정해진다.

## tasks.max와의 관계 — Active-Standby인가 Active-Active인가

- 노드를 2대 올려도 **태스크가 1개면 한 대만 일한다.** 나머지 한 대는 대기 상태로 앉아 있다가 장애 시에만 받는다.

| `tasks.max` | 평상시 | 장애 시 |
| --- | --- | --- |
| 1 | `mm2node1`만 복제, `mm2node2`는 유휴 | 이어받음 (처리량 동일) |
| 2 이상 | 두 노드가 태스크를 나눠 가짐 | 이어받음 (**남은 1대가 전부 떠안음**) |

- [설정 예제 문서](mirror-maker-2-configuration-example.md)의 구성은 `tasks.max = 1`이다. **2대를 올릴 거라면 이 값부터 올려야** 평상시에도 두 대를 쓴다.
- 다만 `tasks.max`를 올릴 때는 **한 대가 죽은 상태에서 남은 한 대가 전부 감당할 수 있는지**를 기준으로 잡아야 한다. 노드당 힙과 버퍼가 그만큼 더 필요하다.
- 태스크 수는 **복제할 토픽-파티션 총 개수**를 넘으면 의미가 없다. 남는 태스크는 논다.

## 확인 방법

```bash
# ① Connect 워커 그룹이 보이는지
/kafka/default/bin/kafka-consumer-groups.sh \
  --bootstrap-server endhost1:9092 --list | grep -i mm2

# ② 그룹에 노드가 몇 대 붙어 있는지 (MEMBER / HOST 열 확인)
/kafka/default/bin/kafka-consumer-groups.sh \
  --bootstrap-server endhost1:9092 --describe --group <위에서 찾은 그룹>

# ③ 내부 토픽이 타깃에 생성됐는지
/kafka/default/bin/kafka-topics.sh \
  --bootstrap-server endhost1:9092 --list | grep '^mm2-'
```

- ②에서 **멤버가 1대만 보이면 두 노드가 그룹을 못 이룬 것**이다. 전제 조건 ①(내부 REST)과 방화벽을 먼저 본다.
- 로그에서는 리밸런스 시작·완료 메시지와 태스크 할당 내역을 확인한다.

```bash
grep -iE 'rebalance|assigned|revok' /kafka/default/logs/mm2.log | tail -50
```

## 장애 조치 타임라인

| 시점 | 일어나는 일 |
| --- | --- |
| T+0 | `mm2node1` 프로세스 종료 |
| ~T+10초 | 세션 타임아웃으로 그룹에서 제외 (`session.timeout.ms`) |
| T+0 ~ T+5분 | **대기** — `scheduled.rebalance.max.delay.ms` 동안 복귀를 기다림 |
| T+5분 | 리밸런스 실행, `mm2node2`가 태스크 인수 |
| 직후 | 마지막 커밋 오프셋부터 복제 재개 (**중복 가능**) |

- 이 5분이 허용 가능한지가 설계 판단의 전부다. CDC 복제라면 보통 감수하지만, 지연에 민감하면 값을 낮추고 리밸런스 빈도를 감수한다.

## 운영 주의

- **두 노드의 설정 파일을 같이 고쳐야 한다.** 한쪽만 바꾸고 재시작하면 ②의 "조용한 증상"이 나온다. 배포를 자동화하거나 `md5sum`으로 확인하는 절차를 둔다.
- 힙 설정(`KAFKA_HEAP_OPTS`)은 **노드마다 따로** 잡는다. 한 대가 죽었을 때 남은 한 대가 태스크를 전부 떠안으므로, 평상시 사용량 기준으로 빠듯하게 잡으면 장애 시 OOM이 난다.
- 롤링 재시작은 **한 대씩, 리밸런스가 끝난 걸 확인하고** 진행한다. 두 대를 동시에 내리면 복제가 완전히 멈춘다.
- 두 노드를 **같은 랙/같은 하이퍼바이저에 두지 않는다.** 함께 죽으면 HA의 의미가 없다.

## 관련 문서

- [MirrorMaker 2 설정 예제](mirror-maker-2-configuration-example.md) — properties 옵션별 설명, 리소스 제한
- [MirrorMaker 2](mirror-maker.md) — 복제 흐름, 배치 위치, exactly-once
- [Kafka Connect 기본 토픽](default-topics.md) — connect-offsets / status / configs
