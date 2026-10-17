# 트랜잭션 · 스케줄 · 비동기 어노테이션

> 작성일: 2026-10-17

- 참조 : [Using @Transactional](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
- 참조 : [Task Execution and Scheduling](https://docs.spring.io/spring-framework/reference/integration/scheduling.html) — `@Scheduled` / `@Async`

- 셋 다 **프록시 기반**이라 self-invocation 함정을 공유한다. 먼저 [프록시와 self-invocation](proxy-and-self-invocation.md)을 보면 아래가 쉽다.
- 공통점은 **"메서드 실행을 둘러싸고 뭔가를 한다"**는 것이다. 트랜잭션은 앞뒤로 커밋/롤백을, 스케줄은 호출 자체를, 비동기는 실행 스레드를 바꾼다.

## 한눈에 보기

| | 목적 | 언제 쓰나 | 활성화 |
| --- | --- | --- | --- |
| `@Transactional` | 여러 DB 작업을 **전부 성공 또는 전부 취소** | 쓰기가 2건 이상 섞이는 서비스 메서드 | Boot 자동 |
| `@Scheduled` | 호출자 없이 **시간이 되면 실행** | 배치, 집계, 캐시 갱신, 헬스체크 | `@EnableScheduling` |
| `@Async` | 호출자를 **기다리게 하지 않음** | 알림·로깅 등 응답과 무관한 작업 | `@EnableAsync` |

## @Transactional

### 무엇을 위한 것인가

- **여러 DB 작업을 하나의 단위로 묶는다.** 중간에 실패하면 앞의 작업까지 되돌려 **반쯤 저장된 상태**를 만들지 않는다.
- 없으면 커넥션을 직접 얻고, `setAutoCommit(false)`, `commit()`, 예외 시 `rollback()`, `finally`에서 `close()`를 메서드마다 써야 한다. 그 코드를 프록시가 대신한다.

### 언제 붙이나

- **쓰기가 2건 이상 일어나는 서비스 메서드** — 주문 저장 + 이력 저장처럼 함께 성공해야 하는 것들
- **읽기 전용 조회** — `readOnly = true`로 최적화 힌트를 준다
- **붙이지 않아도 되는 곳** — 단건 저장만 하는 메서드. Spring Data 리포지토리 메서드는 자체 트랜잭션을 갖는다
- 위치는 **서비스 계층**이 기본이다. 리포지토리에 붙이면 메서드 단위로 쪼개져 묶는 의미가 없고, 컨트롤러에 붙이면 범위가 너무 넓어진다

```java
@Service
public class PaymentService {

    @Transactional
    public void pay(Long orderId) {
        orderRepository.updateStatus(orderId, "PAID");
        historyRepository.insert(orderId);
        // 정상 리턴하면 커밋, RuntimeException이면 둘 다 롤백
    }

    @Transactional(readOnly = true)
    public Order find(Long id) { ... }

    @Transactional(rollbackFor = Exception.class)
    public void importFile(File f) throws IOException { ... }
}
```

### 기본값에서 가장 자주 걸리는 것

- **기본 롤백 대상은 `RuntimeException`과 `Error`뿐이다.** `IOException` 같은 **checked exception은 롤백되지 않는다.**
  - 잡아서 `RuntimeException`으로 감싸거나, `rollbackFor`를 지정한다
  - Spring 6.2부터는 전역으로 `rollbackOn = ALL_EXCEPTIONS` 지정도 가능하다
- **예외를 try-catch로 삼키면 롤백되지 않는다.** 프록시는 메서드 밖으로 나온 예외만 본다.

```java
@Transactional
public void bad() {
    try {
        repository.insert(a);
        repository.insert(b);   // 실패
    } catch (Exception e) {
        log.error("실패", e);   // ❌ 예외를 삼켜서 커밋된다
    }
}
```

- 기본 전파(propagation)는 `REQUIRED`다. 이미 트랜잭션이 있으면 **거기에 참여하고, 없으면 새로 시작**한다.
- 클래스에 붙이면 모든 메서드의 기본값이 되고, 메서드 쪽이 우선한다.
- 프록시 모드에서는 **`public` 메서드 기준으로 생각하는 것이 안전하다.**

### 트랜잭션은 짧게

- 트랜잭션이 열려 있는 동안 **DB 커넥션을 점유한다.** 안에서 외부 API를 호출하거나 파일을 읽으면 그동안 커넥션이 묶인다.
- 커넥션 풀이 마르면 **애플리케이션 전체가 멈춘다.** 무거운 작업은 트랜잭션 밖으로 빼낸다.

## @Scheduled

### 무엇을 위한 것인가

- **호출자 없이 시간이 되면 실행**된다. 요청-응답 흐름과 무관하게 도는 작업을 만든다.
- 외부 cron이나 별도 스케줄러 없이 **애플리케이션 안에서** 해결한다.

### 언제 쓰나

- 주기적 집계·정산, 만료 데이터 정리
- **메모리 캐시 갱신** — DB의 정책/설정을 주기적으로 다시 읽어오는 패턴
- 외부 시스템 헬스체크, 지표 수집
- **안 맞는 경우** — 실행 이력·재시도·실패 알림이 필요한 본격 배치. 그건 Airflow 같은 워크플로 도구나 Spring Batch가 맞다

```java
@Configuration
@EnableScheduling                       // 필수. 없으면 한 번도 실행되지 않는다
public class SchedulerConfig { }
```

```java
@Component
public class ReportJob {

    @Scheduled(fixedDelay = 60_000)     // 이전 실행 "종료" 후 60초
    public void collect() { ... }

    @Scheduled(fixedRate = 60_000)      // 이전 실행 "시작" 후 60초
    public void ping() { ... }

    @Scheduled(cron = "0 0 3 * * *")    // 매일 03:00 (초 분 시 일 월 요일)
    public void daily() { ... }
}
```

### fixedDelay vs fixedRate vs cron

| 속성 | 기준 | 고르는 기준 |
| --- | --- | --- |
| `fixedDelay` | 이전 실행이 **끝난** 시점부터 | 겹치면 안 되는 작업 |
| `fixedRate` | 이전 실행이 **시작된** 시점부터 | 일정한 간격이 중요한 작업 |
| `cron` | 시각 지정 | "매일 새벽 3시"처럼 시각이 정해진 작업 |

- `cron`은 **필드가 6개**(초부터)로 리눅스 crontab(5개)과 다르다.
- `fixedRate`는 작업이 주기보다 오래 걸리면 **다음 실행이 밀린다.** 기본 스레드가 1개라 더 그렇다.

### 스레드 풀이 기본 1개

- **스케줄 작업이 여러 개면 서로를 기다린다.** 하나가 오래 걸리면 다른 작업이 통째로 밀린다.

```yaml
spring:
  task:
    scheduling:
      pool:
        size: 5
```

- 메서드는 **인자가 없어야 하고 반환값은 무시**된다.

### 서버를 여러 대 띄우면

- **모든 인스턴스에서 동시에 실행된다.** 집계가 중복되거나 같은 행을 여러 번 갱신한다.
- 한 번만 돌아야 하면 분산 락(ShedLock 등)이 따로 필요하다. 읽기 전용 캐시 갱신처럼 **중복이 무해한 작업**이면 그대로 둬도 된다.

## @Async

### 무엇을 위한 것인가

- **호출자를 기다리게 하지 않는다.** 메서드가 즉시 리턴하고 실제 작업은 다른 스레드에서 돈다.
- 응답 시간에서 **부가 작업을 들어낸다.** 주문 처리 200ms + 알림 발송 800ms가 1초였다면, 알림을 비동기로 돌려 200ms로 만든다.

### 언제 쓰나

- 메일·푸시 알림, 외부 시스템 통지
- 감사 로그 적재, 통계 전송
- **쓰면 안 되는 경우** — 결과가 응답에 필요한 작업, **유실되면 안 되는 작업.** 애플리케이션이 죽으면 큐에 남은 작업은 그대로 사라진다. 보장이 필요하면 Kafka 같은 메시지 큐를 쓴다

```java
@Configuration
@EnableAsync                            // 필수
public class AsyncConfig { }
```

```java
@Service
public class NotificationService {

    @Async
    public void send(String to) {       // 호출 즉시 리턴
        mailClient.send(to);
    }

    @Async
    public CompletableFuture<Report> build(Long id) {   // 결과가 필요하면
        return CompletableFuture.completedFuture(generate(id));
    }
}
```

### 반환 타입과 예외 유실

- **반환 타입은 `void` 또는 `CompletableFuture`여야 한다.** 그냥 `Report`로 두면 값을 기다려야 하므로 어노테이션이 무의미해진다.
- **`void`면 예외가 사라진다.** 호출자는 이미 리턴한 뒤라 받을 방법이 없다. 로그를 남기려면 `AsyncUncaughtExceptionHandler`를 등록한다.

```yaml
spring:
  task:
    execution:
      pool:
        core-size: 8
        max-size: 16
        queue-capacity: 100
```

- 큐가 가득 차면 거부 정책에 따라 예외가 나거나 호출 스레드에서 실행된다. **기본값으로 쓰다가 트래픽이 몰리면 터지는 지점**이다.

## 조합할 때

### @Transactional + @Async

```java
@Transactional
public void outer() {
    asyncService.doWork();   // ❌ 트랜잭션이 따라가지 않는다
}
```

- 트랜잭션은 **`ThreadLocal`에 보관**된다. `@Async`가 스레드를 바꾸는 순간 끊긴다.
- 비동기 메서드 쪽에 `@Transactional`을 따로 붙여 **별도 트랜잭션**으로 처리해야 한다.
- 더 큰 문제는 **커밋 전에 비동기 작업이 먼저 돌 수 있다는 것**이다. 아직 커밋 안 된 데이터를 다른 스레드가 조회하면 못 찾는다. 커밋 이후에 실행하려면 `@TransactionalEventListener(phase = AFTER_COMMIT)`를 쓴다.

### @Scheduled + @Transactional

- 같이 붙여도 **동작한다.** 스케줄러가 외부에서 프록시를 통해 호출하므로 self-invocation이 아니다.
- 다만 스케줄 작업은 길어지기 쉬워 **트랜잭션을 오래 쥐고 있게 된다.** 조회와 쓰기를 나눠 범위를 좁힌다.

## 관련 문서

- [프록시와 self-invocation](proxy-and-self-invocation.md) — 왜 안 먹는가
- [캐시 · 생명주기 어노테이션](cache-and-lifecycle-annotations.md)
- [Datasource와 DSLContext](datasource_dslcontext.md) — 트랜잭션이 걸리는 DataSource 구성
