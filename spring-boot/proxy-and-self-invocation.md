# 프록시와 self-invocation

> 작성일: 2026-10-17

- 참조 : [Proxying Mechanisms](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)

- **왜 알아야 하나** — 트랜잭션·캐시·비동기 같은 **부가 기능을 비즈니스 코드에서 분리**하는 장치가 프록시다. `@Transactional` 한 줄로 트랜잭션 관리 코드가 사라지는 것이 그 결과다.
- **언제 떠올려야 하나** — "분명 어노테이션을 붙였는데 안 먹는다" 싶을 때. **에러가 나지 않고 조용히 무시되므로**, 구조를 아는 것이 유일한 방어다.

## 프록시가 가로챈다는 것

```
[프록시 계열]
   호출자 ──▶ 프록시 ──▶ 실제 객체
                │
                └─ 여기서 부가 기능 수행
                   (트랜잭션 시작 / 캐시 조회 / 스레드 전환 / 권한 검사)

[비프록시]
   컨테이너 ──▶ 실제 객체의 메서드를 리플렉션으로 직접 호출
                가로채는 단계가 없다
```

- Spring은 빈을 등록할 때 **원본 객체를 감싼 대리 객체(프록시)**를 만들어 컨테이너에 넣는다.
- `@Autowired`로 주입받는 것은 **실제 객체가 아니라 이 프록시**다. 호출하면 프록시가 먼저 받아 부가 기능을 수행하고, 그다음 실제 객체로 넘긴다.
- **목적은 관심사의 분리다.** 트랜잭션 커밋/롤백, 캐시 조회/저장 같은 반복 코드를 비즈니스 메서드 밖으로 빼낸다.
- **대가는 "호출 경로에 의존한다"는 것**이다. 프록시를 통과하지 않는 경로로 호출하면 부가 기능이 통째로 사라진다. 아래 함정이 전부 여기서 나온다.

## 활성화 어노테이션 — @Enable*

**프록시를 만들어 줄 처리기가 등록돼야 어노테이션이 의미를 갖는다.** 자바 어노테이션 자체는 아무 동작이 없는 메타데이터일 뿐이다.

```
@Scheduled(fixedRate = 60000)        ← "1분마다 돌려줘" 라고 써 붙인 표시

   ScheduledAnnotationBeanPostProcessor   ← 이걸 읽고 실제로 등록하는 주체
              ▲
   @EnableScheduling                      ← 이게 있어야 위 처리기가 빈이 된다
```

| 기능 어노테이션 | 켜는 어노테이션 | Boot 자동 |
| --- | --- | --- |
| `@Transactional` | `@EnableTransactionManagement` | **✅ 자동** |
| `@Cacheable` `@CacheEvict` | `@EnableCaching` | ❌ 직접 |
| `@Scheduled` | `@EnableScheduling` | ❌ 직접 |
| `@Async` | `@EnableAsync` | ❌ 직접 |

- **Boot가 자동으로 켜주는 건 트랜잭션뿐이다.** `TransactionAutoConfiguration`이 대신 선언해 준다.
- 그래서 "어노테이션만 붙이면 되네?"라는 경험이 생기고, 캐싱·스케줄링·비동기에서 그대로 당한다.
- 의도치 않게 백그라운드 스레드가 도는 걸 막기 위한 설계라 **자동화하지 않는 쪽이 정상**이다.

```java
@EnableCaching
@EnableScheduling                   // 빠뜨리면 @Scheduled가 한 번도 실행되지 않는다
@SpringBootApplication
public class MyApplication { }
```

- 메인 클래스에 **애플리케이션당 한 번**만 선언한다. 서비스 클래스마다 붙일 필요 없다.

## 공통 함정

### ① self-invocation — 같은 클래스 안에서의 호출

```java
@Service
public class OrderService {

    public void process(Order order) {
        validate(order);
        save(order);        // ❌ this.save() — 프록시를 안 거친다
    }

    @Transactional
    public void save(Order order) {
        repository.insert(order);   // 트랜잭션 없이 실행됨
    }
}
```

- 공식문서의 설명 그대로다. 호출이 실제 객체에 도달한 뒤의 `this.bar()`는 **프록시가 아닌 `this` 참조로** 실행되므로 부가 기능을 건너뛴다.
- **해결 순서** (공식문서 권장 순)
  1. **구조를 바꿔 self-invocation을 없앤다** — `save()`를 다른 빈으로 분리하는 것이 가장 깔끔하다
  2. 자기 자신을 주입받아 호출한다
  3. `AopContext.currentProxy()` — 최후 수단

```java
@Service
public class OrderService {

    private final OrderService self;   // ② 자기 자신 주입

    public OrderService(@Lazy OrderService self) {
        this.self = self;
    }

    public void process(Order order) {
        self.save(order);    // ✅ 프록시를 거친다
    }
}
```

### ② private / final 메서드

- CGLIB 프록시는 **상속으로 메서드를 오버라이드**해 가로챈다. 그래서 오버라이드할 수 없는 것은 가로챌 수 없다.
  - `private` 메서드 — 불가
  - `final` 메서드 — 불가
  - `final` 클래스 — 프록시 생성 자체가 불가
- **Kotlin에서 특히 걸린다.** 클래스와 메서드가 기본 `final`이라 `open`을 붙이거나 `all-open` 플러그인을 써야 한다.

### ③ 증상이 조용하다

- 셋 다 **예외가 발생하지 않는다.** 트랜잭션이 안 걸린 채로, 캐시가 안 걸린 채로 정상 동작한다.
- 그래서 **운영에서 데이터가 깨진 뒤에 발견**된다. 롤백이 안 돼 중간까지만 저장된 데이터, 1분마다 갱신된다고 믿었는데 기동 시점 그대로인 캐시 같은 것들이다.

### 프록시인지 확인하는 법

```java
@Service
public class DiagnosticService {

    private final OrderService orderService;

    @PostConstruct
    public void check() {
        log.info("class = {}", orderService.getClass().getName());
        log.info("isProxy = {}", AopUtils.isAopProxy(orderService));
    }
}
```

```
class = com.example.OrderService$$SpringCGLIB$$0     ← 프록시 (정상)
class = com.example.OrderService                     ← 원본 (어노테이션이 안 먹는 상태)
```

- 클래스명에 `$$SpringCGLIB$$`나 `$Proxy`가 붙어 있으면 프록시다.
- `@Enable*`을 빠뜨렸거나 해당 클래스에 가로챌 어노테이션이 하나도 없으면 원본이 그대로 등록된다.

## 이 구조를 쓰는 어노테이션

**아래는 전부 위 함정을 공유한다.** 하나를 이해하면 나머지도 같다.

| 어노테이션 | 가로채서 하는 일 |
| --- | --- |
| `@Transactional` | 트랜잭션 시작 / 커밋 / 롤백 |
| `@Cacheable` `@CacheEvict` `@CachePut` | 캐시 조회 / 저장 / 삭제 |
| `@Async` | 별도 스레드에서 실행 |
| `@Scheduled` | 주기 실행 등록 |
| `@KafkaListener` | 컨슈머 생성, poll 루프, 재시도 |
| `@PreAuthorize` `@Secured` | 권한 검사 |
| `@Validated` | 파라미터 검증 |
| `@Retryable` | 실패 시 재시도 (spring-retry) |

- 반대로 **`@PostConstruct` / `@PreDestroy`는 프록시가 아니다.** 컨테이너가 직접 호출하므로 이 함정이 없다. → [캐시 · 생명주기 어노테이션](cache-and-lifecycle-annotations.md)

## 관련 문서

- [트랜잭션 · 스케줄 · 비동기 어노테이션](transaction-schedule-async-annotations.md)
- [캐시 · 생명주기 어노테이션](cache-and-lifecycle-annotations.md)
- [@Bean](bean.md) — 빈 등록과 자동 설정 back off
