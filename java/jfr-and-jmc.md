# JFR과 JMC

> 작성일: 2026-10-06

- JFR(Java Flight Recorder)과 JMC(JDK Mission Control)는 한 쌍이다. **JFR이 기록하고, JMC가 본다.**
- 비행기로 치면 JFR이 블랙박스, JMC가 판독 장비다. 이름이 그래서 Flight Recorder / Mission Control이다.

| | 이름 | 역할 | 어디 있나 |
| --- | --- | --- | --- |
| JFR | Java Flight Recorder | JVM 내부에서 이벤트를 수집해 `.jfr` 파일로 남김 | JDK 내장 (설치 불필요) |
| JMC | JDK Mission Control | `.jfr` 파일을 열어 그래프·분석으로 표시 | 별도 다운로드하는 GUI 앱 |

```plain text
운영 서버                              로컬 PC
┌──────────────────┐
│ JVM              │
│  └─ JFR 기록 중   │ ──── app.jfr 파일 ────→  JMC로 열어서 분석
│     (오버헤드 1~2%)│         (scp 등으로 복사)
└──────────────────┘
```

- **서버에는 JMC를 깔 필요가 없다.** JDK만 있으면 기록이 되고, 파일만 받아 로컬에서 열면 된다.
- JDK 11부터 JFR이 OpenJDK에 포함되어 상용 라이선스 없이 쓸 수 있다. (Oracle JDK 8 시절에는 상용 기능이었음)

## 기록 시작하기

- 애플리케이션 시작 시 옵션으로 켜기

```bash
java -XX:StartFlightRecording=duration=120s,filename=app.jfr,settings=profile -jar app.jar
```

- 이미 떠 있는 프로세스에 붙이기 (운영 중 장애 분석에 주로 이걸 쓴다)

```bash
jcmd <pid>                                              # pid 찾기
jcmd <pid> JFR.start duration=120s filename=/tmp/app.jfr settings=profile
jcmd <pid> JFR.check                                    # 진행 상황 확인
jcmd <pid> JFR.dump name=1 filename=/tmp/app.jfr        # 중간에 덤프
jcmd <pid> JFR.stop name=1                              # 중지
```

- `settings` 프로필 2종
	- `default` : 오버헤드 약 1% 미만. 상시 켜둬도 되는 수준
	- `profile` : 오버헤드 약 2%. 메서드 샘플링이 촘촘해져 핫스팟 분석에 적합. 보통 이걸 쓴다

## JMC 없이 터미널에서 보기

- 급할 때나 GUI를 못 띄우는 환경에서는 JDK에 들어있는 `jfr` 명령으로 충분히 본다.

```bash
jfr summary app.jfr                                     # 이벤트 종류별 건수 요약
jfr print --events GCPhasePause app.jfr                 # 특정 이벤트만 출력
jfr print --events ExecutionSample app.jfr | head -50   # 메서드 샘플
jfr print --json --events ObjectAllocationSample app.jfr
```

## 무엇을 볼 수 있나

- **GC** : 수집 횟수, Pause 시간, 세대별 추이 → Full GC가 잦은지, Pause가 긴지
- **힙 / 메모리** : 사용량 추이, 할당이 많은 클래스와 호출 지점 → OOM 원인 추적
- **CPU / 핫 메서드** : 샘플링 기반으로 어느 메서드가 CPU를 먹는지
- **락 경합(Monitor Blocked)** : 어떤 스레드가 무엇을 기다리며 막혀 있는지
- **I/O** : 느린 파일·소켓 읽기/쓰기
- **스레드 덤프 / 예외 발생 추이**

## 실무에서 쓰는 지점

- Spark Executor OOM, Flink TaskManager 메모리 이슈처럼 **"왜 메모리가 차는지"**를 봐야 할 때 : 할당 샘플(ObjectAllocationSample)에서 할당 지점을 역추적한다.
- GC Pause가 길어 heartbeat가 끊길 때 : GC 이벤트와 Pause 구간을 타임라인으로 확인한다.
- 처리량이 안 나올 때 : 핫 메서드와 락 경합 중 어느 쪽이 병목인지 가른다.
- 재현이 어려운 간헐적 장애 : `default` 프로필로 상시 기록을 돌려두고, 증상 발생 직후 `JFR.dump`로 그 구간을 떠낸다.

## 주의

- 컨테이너 환경에서는 `.jfr` 파일을 **컨테이너 밖 볼륨 경로에 쓰거나** 즉시 복사해야 한다. 파드가 죽으면 파일도 같이 사라진다.
- `duration` 없이 시작하면 멈출 때까지 계속 기록한다. 디스크가 찰 수 있으니 `maxsize`, `maxage`를 같이 준다.

```bash
jcmd <pid> JFR.start name=always-on maxsize=200m maxage=1h settings=default
```

- JMC는 실행 중인 JVM에 직접 붙어 MBean을 보는 JMX 콘솔 기능도 있지만, 실무에서는 대부분 `.jfr` 파일 분석 용도로 쓴다.
