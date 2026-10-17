# 캐시 · 생명주기 어노테이션

> 작성일: 2026-10-17

- 참조 : [Declarative Annotation-based Caching](https://docs.spring.io/spring-framework/reference/integration/cache/annotations.html)
- 참조 : [Customizing the Nature of a Bean](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html)

- 두 주제를 같이 두는 이유는 **"데이터를 언제 메모리에 올려두는가"**가 겹치기 때문이다. `@Cacheable`은 호출 시점에, `@PostConstruct`는 기동 시점에 올린다.
- 동작 방식은 반대다. **캐시는 프록시 기반**이라 함정이 있고, **생명주기 콜백은 컨테이너가 직접 호출**해 함정이 없다. → [프록시와 self-invocation](proxy-and-self-invocation.md)

## @Cacheable / @CacheEvict / @CachePut

### 무엇을 위한 것인가

- **같은 입력에 같은 결과가 나오는 조회를 반복하지 않는다.** 두 번째 호출부터 메서드를 아예 실행하지 않고 저장해 둔 값을 돌려준다.
- 호출하는 쪽도, 호출되는 메서드 본문도 **캐시를 모른다.** 캐시를 걷어내도 코드가 그대로 동작하는 것이 이 방식의 이점이다.

### 언제 쓰나

- **자주 읽고 거의 안 바뀌는 데이터** — 코드 테이블, 정책/설정, 카테고리 목록
- 비용이 큰 조회 — 조인이 많거나 외부 API를 타는 것
- **안 맞는 경우**
  - 매번 달라지는 데이터. 캐시 히트가 없어 관리 비용만 남는다
  - **쓰기 메서드.** `@Cacheable`을 붙이면 두 번째 호출부터 DB에 안 간다
  - 사용자마다 결과가 다른데 키에 사용자를 안 넣는 경우. **남의 데이터가 보인다**

```java
@Configuration
@EnableCaching                          // 필수
public class CacheConfig { }
```

```java
@Service
public class ProductService {

    @Cacheable(cacheNames = "product", key = "#id")
    public Product find(Long id) {
        return repository.findById(id).orElseThrow();   // 캐시에 있으면 호출조차 안 됨
    }

    @Cacheable(cacheNames = "product", key = "#id", unless = "#result == null")
    public Product findNullable(Long id) { ... }        // null은 캐싱하지 않음

    @CacheEvict(cacheNames = "product", key = "#product.id")
    public void update(Product product) { ... }         // 정상 종료 시 해당 키 삭제

    @CacheEvict(cacheNames = "product", allEntries = true)
    public void reloadAll() { ... }                     // 전체 삭제

    @CachePut(cacheNames = "product", key = "#product.id")
    public Product save(Product product) {              // 항상 실행하고 결과로 갱신
        return repository.save(product);
    }
}
```

### 셋의 차이

| 어노테이션 | 메서드 실행 | 캐시 | 쓰는 자리 |
| --- | --- | --- | --- |
| `@Cacheable` | 캐시에 있으면 **건너뜀** | 없을 때만 저장 | 조회 |
| `@CachePut` | **항상 실행** | 항상 갱신 | 수정 (결과를 캐시에 반영) |
| `@CacheEvict` | 항상 실행 | 삭제 | 수정·삭제 (캐시를 버림) |

- `@Cacheable`과 `@CachePut`의 차이가 핵심이다. **수정 메서드에 `@Cacheable`을 쓰면 안 된다.**
- 수정 후 처리는 보통 **`@CacheEvict`로 버리는 쪽**이 안전하다. `@CachePut`은 반환값이 DB 최종 상태와 정확히 같아야 한다.

### key, condition, unless

- `key`를 생략하면 메서드 인자로 키를 만든다. 인자가 여럿이면 의도와 다를 수 있으니 **SpEL로 명시**하는 편이 안전하다.
- `condition`은 **실행 전**, `unless`는 **실행 후 결과**로 판단한다. `#result`는 `unless`에서만 쓸 수 있다.

```java
@Cacheable(cacheNames = "product",
           key = "#id",
           condition = "#id > 0",            // 실행 전 판단
           unless = "#result == null")       // 실행 후 결과로 판단
```

### @CacheEvict의 삭제 시점

- 기본은 **메서드가 정상 종료한 뒤**다. 예외가 나면 삭제되지 않는다.
- `beforeInvocation = true`로 두면 **실행 전에 삭제**한다. 메서드 성공 여부와 무관하게 비워야 할 때 쓴다.

## @PostConstruct / @PreDestroy

### 무엇을 위한 것인가

- **생성자로는 할 수 없는 초기화**를 하는 자리다. 생성자 시점에는 의존성 주입이 끝나지 않았을 수 있다.
- `@PostConstruct`는 **모든 주입이 끝난 뒤 딱 한 번**, `@PreDestroy`는 **컨테이너 종료 직전**에 실행된다.

### 언제 쓰나

- **설정값 검증** — 필수 프로퍼티가 비어 있으면 기동을 실패시킨다. 운영 중 NPE로 발견하는 것보다 낫다
- 간단한 자료구조 준비, 외부 클라이언트 초기 설정
- `@PreDestroy`는 **직접 만든 스레드·커넥션 정리**. 라이브러리 객체는 `close()`가 있으면 Spring이 알아서 부르므로 보통 필요 없다

```java
@Service
public class ConfigValidator {

    private final AppProperties props;
    private Map<String, String> codeTable;

    public ConfigValidator(AppProperties props) {
        this.props = props;         // ① 생성자 — 아직 주입이 다 안 끝났을 수 있다
    }

    @PostConstruct
    public void init() {            // ② 모든 주입 완료 후 1회
        if (props.getEndpoint() == null) {
            throw new IllegalStateException("endpoint 설정 누락");
        }
        this.codeTable = loadCodes();
    }

    @PreDestroy
    public void cleanup() {         // ③ 컨테이너 종료 직전
        codeTable.clear();
    }
}
```

- `jakarta.annotation` 패키지다. **Spring에 종속되지 않아** 공식문서도 `InitializingBean` 구현보다 이쪽을 권한다.
- 반환값이 없고 인자를 받지 않는다. **프록시를 안 거치므로 self-invocation 함정이 없다.**

### 초기화 수단이 겹칠 때의 순서

1. `@PostConstruct`
2. `InitializingBean.afterPropertiesSet()`
3. `@Bean(initMethod = "...")`로 지정한 메서드

- 소멸도 같은 패턴이다 — `@PreDestroy` → `DisposableBean.destroy()` → `destroyMethod`.
- 같은 메서드를 여러 방식으로 지정하면 **한 번만 실행된다.**

### 무거운 작업을 넣으면 안 된다

- **초기화 메서드는 싱글톤 생성 락 안에서 실행된다.** 외부 호출이나 대량 적재를 여기서 하면 기동 전체가 그만큼 지연되고, 실패하면 애플리케이션이 아예 뜨지 않는다.
- 설정 검증과 자료구조 준비 정도만 한다. 무거운 작업은 아래로 옮긴다.

## 기동이 끝난 뒤에 실행하려면

`@PostConstruct`는 **그 빈 하나만 준비된 시점**이다. 다른 빈이나 웹 서버가 아직 준비되지 않았을 수 있다.

| 수단 | 시점 | 쓰는 경우 |
| --- | --- | --- |
| `@PostConstruct` | 그 빈만 준비됨 | 설정 검증, 가벼운 준비 |
| `SmartInitializingSingleton` | 모든 싱글톤 생성 완료 | 다른 빈들을 훑어야 할 때 |
| `ApplicationRunner` | 컨텍스트 준비 완료 | **기동 인자**를 받아 처리할 때 |
| `@EventListener(ApplicationReadyEvent.class)` | 웹 서버 포트까지 열림 | **외부 호출, 캐시 워밍** |

```java
@Component
public class CacheWarmer {

    private final ProductService productService;

    @EventListener(ApplicationReadyEvent.class)
    public void warmUp() {
        productService.find(1L);    // ✅ 다른 빈이므로 프록시를 거친다 → 캐시 적재됨
    }
}
```

- **외부 호출이나 캐시 워밍은 `ApplicationReadyEvent`가 기본값**이다. 기동을 막지 않고, 실패해도 애플리케이션은 떠 있다.
- `ApplicationRunner`와 `CommandLineRunner`는 거의 같다. 인자를 파싱된 형태로 받느냐(`ApplicationArguments`), 원본 `String[]`으로 받느냐의 차이다.

## 직접 만드는 캐시

`@Cacheable`이 안 맞는 경우가 있다. **전체를 통째로 들고 있어야 하거나, 주기적으로 통째로 갈아끼워야 할 때**다.

```java
@Service
@RequiredArgsConstructor
public class PolicyCacheService {

    private final PolicyRepository repository;

    private volatile Map<Long, Policy> cache = Collections.emptyMap();   // ① volatile

    @EventListener(ApplicationReadyEvent.class)
    public void init() {
        refresh();                                   // ② 기동 후 최초 적재
    }

    @Scheduled(fixedRate = 60_000)
    public void scheduledRefresh() {
        refresh();                                   // ③ 1분마다 갱신
    }

    private void refresh() {
        try {
            Map<Long, Policy> newCache = repository.findAllActive().stream()
                    .collect(Collectors.toMap(Policy::getId, p -> p));
            this.cache = newCache;                   // ④ 참조 한 번으로 원자적 교체
            log.info("policy cache refreshed: {}", newCache.size());
        } catch (Exception e) {
            log.error("refresh 실패 — 기존 캐시 유지", e);   // ⑤ 실패해도 서비스는 계속
        }
    }

    public Policy get(Long id) {
        return cache.get(id);
    }
}
```

| | `@Cacheable` | 직접 만들기 |
| --- | --- | --- |
| 적재 | 호출될 때 하나씩 (lazy) | 기동 시 통째로 (eager) |
| 갱신 | `@CacheEvict` 또는 TTL | `@Scheduled`로 주기 교체 |
| 첫 호출 | 느림 | 빠름 |
| 전체 순회 | 불가 | 가능 |
| 맞는 경우 | 키 단위 조회 | **건수가 적고 전체를 다 쓰는 메타데이터** |

- **`volatile` + 통째 교체**가 요령이다. 기존 맵을 수정하지 않고 새 맵을 만들어 참조만 바꾸면, 읽는 쪽은 락 없이 항상 일관된 스냅샷을 본다.
- **갱신 실패 시 기존 캐시를 유지**하는 것이 중요하다. 예외를 삼키고 로그만 남기면 일시적 DB 장애에도 서비스가 계속된다.
- `@Scheduled`를 쓰므로 **`@EnableScheduling`이 필요하다.** 빠뜨리면 기동 시 1회 적재되고 영원히 갱신되지 않는다. 에러도 나지 않는다.

## 섞을 때 주의

### @PostConstruct 안에서 @Cacheable 호출

```java
@PostConstruct
public void warmUp() {
    find(1L);          // ❌ @Cacheable 안 먹음 — self-invocation
}
```

- 프록시 함정 그대로다. 게다가 **프록시가 아직 완성되지 않은 시점**일 수도 있다.
- 캐시 워밍은 `ApplicationReadyEvent`에서 **다른 빈을 통해** 호출한다. (위 `CacheWarmer` 예제)

### 캐시 + 트랜잭션

- `@CachePut`이나 `@Cacheable`은 **커밋 전에 캐시에 쓴다.** 이후 롤백되면 **DB에는 없는 값이 캐시에 남는다.**
- 트랜잭션 커밋 이후로 미루려면 `TransactionAwareCacheManagerProxy`로 CacheManager를 감싼다.

## 관련 문서

- [프록시와 self-invocation](proxy-and-self-invocation.md) — 캐시가 안 먹는 이유
- [트랜잭션 · 스케줄 · 비동기 어노테이션](transaction-schedule-async-annotations.md) — `@Scheduled`, `@EnableScheduling`
- [@Bean](bean.md) — `initMethod` / `destroyMethod`, 스코프
