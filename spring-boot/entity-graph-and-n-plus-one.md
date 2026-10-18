# @EntityGraph와 N+1 문제

> 작성일: 2026-10-17

- 참조 : [Configuring Fetch- and LoadGraphs](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html#jpa.entity-graph) — Spring Data JPA
- 참조 : [NamedEntityGraph](https://jakarta.ee/specifications/persistence/3.2/apidocs/jakarta.persistence/jakarta/persistence/NamedEntityGraph.html) — Jakarta Persistence
- 참조 : [Hibernate ORM User Guide](https://docs.hibernate.org/orm/6.6/userguide/html_single/Hibernate_User_Guide.html)

- `@EntityGraph`는 **"이 쿼리를 실행할 때 연관 엔티티를 어디까지 한 번에 같이 끌어와라"**를 지정하는 어노테이션이다.
- 목적은 둘이다. **N+1 쿼리를 막는 것**과, **트랜잭션 밖에서 읽어도 안전하게 만드는 것.**
- `FetchType.LAZY` 선언은 그대로 두고 **쿼리 단위로** 제어한다는 점이 핵심이다.

## 안 쓸 때 — N+1

```java
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)   // ← LAZY
    private Member member;
}
```

```java
public interface OrderRepository extends JpaRepository<Order, Long> { }
```

```java
@Transactional(readOnly = true)
public void report() {
    List<Order> orders = orderRepository.findAll();   // 쿼리 1번
    for (Order o : orders) {
        log.info(o.getMember().getName());            // ← 여기서 1건마다 쿼리
    }
}
```

**실제로 나가는 SQL**

```sql
SELECT * FROM orders;                    -- 1번
SELECT * FROM member WHERE id = 1;       -- +1
SELECT * FROM member WHERE id = 2;       -- +2
SELECT * FROM member WHERE id = 3;       -- +3
...                                      -- 주문 100건이면 총 101번
```

- **쿼리 1번 + 결과 N건마다 1번 = N+1.** 코드상으로는 `findAll()` 한 줄이라 **눈에 보이지 않는다.**
- 개발 DB에서 10건으로 테스트하면 멀쩡하고, 운영에서 10만 건이 되면 그때 드러난다.

## 쓸 때

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    @EntityGraph(attributePaths = "member")
    List<Order> findAll();
}
```

**나가는 SQL**

```sql
SELECT o.*, m.*
  FROM orders o
  LEFT JOIN member m ON o.member_id = m.id;   -- 1번 끝
```

- 엔티티의 `LAZY` 선언은 **그대로 둔다.** 이 메서드로 조회할 때만 같이 가져온다.
- 엔티티를 `EAGER`로 바꾸면 **모든 조회에 영향**이 간다. 목록만 필요한 화면에서도 연관을 다 끌고 온다. `@EntityGraph`는 그런 부작용이 없다.

## 문법

```java
@EntityGraph(
    attributePaths = { "경로1", "경로2.중첩" },   // 같이 가져올 연관 필드
    type = EntityGraphType.FETCH                 // 생략 가능 (기본값)
)
```

| 속성 | 의미 |
| --- | --- |
| `attributePaths` | `String[]`. **엔티티의 필드명**(컬럼명 아님). `.`으로 중첩 경로 |
| `type` | `FETCH`(기본) = 명시한 것만 EAGER, **나머지는 전부 LAZY**<br>`LOAD` = 명시한 것은 EAGER, 나머지는 **엔티티 선언대로** |
| `value` | `@NamedEntityGraph`로 미리 정의한 그래프 이름을 참조할 때 |

- 경로가 하나면 중괄호를 생략할 수 있다.

```java
@EntityGraph(attributePaths = "member")
```

- **`attributePaths`에 쓰는 것은 자바 필드명이다.** `member_id` 같은 컬럼명을 쓰면 기동 시점에 실패한다.
- `FETCH`와 `LOAD`의 차이는 **명시하지 않은 연관을 어떻게 볼 것인가**다. 대부분 기본값 `FETCH`로 충분하다.

## 중첩 경로

`.`으로 몇 단계든 타고 들어간다.

```java
@Entity
public class Order {
    @Id Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    private Member member;

    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;
}

@Entity
public class OrderItem {
    @ManyToOne(fetch = FetchType.LAZY)
    private Product product;
}

@Entity
public class Product {
    @ManyToOne(fetch = FetchType.LAZY)
    private Category category;
}
```

```java
@EntityGraph(attributePaths = {
        "member",                    // Order.member
        "items",                     // Order.items (List)
        "items.product",             // OrderItem.product
        "items.product.category"     // Product.category
})
List<Order> findByStatus(String status);
```

- 타고 들어가는 그래프

```
Order
 ├─ member                      → member
 └─ items                       → order_item   (여러 건)
      └─ product                → product
           └─ category          → category
```

- **4단계를 전부 명시했으니 JOIN 한 방으로 끝난다.** 안 썼다면 주문 100건 × 품목 5개마다 product·category 조회가 따로 나간다.
- **중간 단계를 빠뜨리면 안 된다.** `"items.product"`만 쓰고 `"items"`를 빼도 동작하지만, 경로를 전부 적어두는 편이 의도가 분명하다.

## 언제 쓰나

### ① 목록 조회에서 연관을 함께 쓸 때

- 가장 흔한 경우다. 목록을 돌면서 `o.getMember().getName()` 같은 접근이 있으면 **무조건 N+1이다.**
- 반대로 **연관을 안 쓰는 조회에는 붙이지 않는다.** 쓸데없이 JOIN해서 데이터만 더 끌어온다.

### ② 트랜잭션 밖에서 읽을 때

**이쪽이 더 치명적이다.** 조회 결과를 메모리에 담아두고 나중에 꺼내 쓰는 패턴에서 터진다.

```java
@Service
public class PolicyCacheService {

    private volatile Map<Long, Policy> cache = Collections.emptyMap();

    @EventListener(ApplicationReadyEvent.class)
    public void init() {
        List<Policy> list = repository.findAllActive();   // 여기서 트랜잭션 끝
        this.cache = list.stream().collect(toMap(Policy::getId, p -> p));
    }

    public String getMetricName(Long id) {
        return cache.get(id)
                    .getConditions().get(0)
                    .getMetric().getName();   // ❌ LazyInitializationException
    }
}
```

- 영속성 컨텍스트가 닫힌 뒤에 LAZY 프록시를 건드리면 **`LazyInitializationException`**이 난다.
- `@EntityGraph`로 **미리 다 채워두면** 안전하다. 이후로는 평범한 자바 객체일 뿐이다.

```java
@EntityGraph(attributePaths = {"conditions", "conditions.metric"})
List<Policy> findAllActive();
```

- `spring.jpa.open-in-view`는 **웹 요청에만** 적용된다. 스케줄러 스레드나 기동 시점 초기화에는 도움이 안 된다. → [캐시 · 생명주기 어노테이션](cache-and-lifecycle-annotations.md)

### 안 쓰는 게 나은 경우

- 연관을 **전혀 안 건드리는** 조회
- 단건 조회 — N+1이 아니라 1+1이라 이득이 작다
- 연관 데이터가 **매우 큰** 경우. JOIN 결과 행이 폭증한다

## @NamedEntityGraph — 이름 붙여 재사용

같은 그래프를 여러 메서드에서 쓰면 엔티티에 정의해두고 이름으로 참조한다.

```java
@Entity
@NamedEntityGraph(
    name = "Order.withMemberAndItems",
    attributeNodes = {
        @NamedAttributeNode("member"),
        @NamedAttributeNode(value = "items", subgraph = "items")
    },
    subgraphs = @NamedSubgraph(
        name = "items",
        attributeNodes = @NamedAttributeNode("product"))
)
public class Order { ... }
```

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    @EntityGraph(value = "Order.withMemberAndItems", type = EntityGraphType.LOAD)
    List<Order> findByStatus(String status);
}
```

- 중첩은 `subgraphs`로 표현해야 해서 **`attributePaths`보다 장황하다.**
- 한 군데서만 쓸 거면 `attributePaths`가 낫다. **재사용이 실제로 일어날 때만** 이름을 붙인다.

## 주의할 것

### 컬렉션을 두 개 이상 fetch하면 터진다

```java
// ❌ List 필드 두 개를 동시에
@EntityGraph(attributePaths = {"items", "coupons"})
List<Order> findAll();
```

- `MultipleBagFetchException`이 나거나, 타입에 따라 **카테시안 곱으로 행이 폭증**한다. 품목 5개 × 쿠폰 3개면 주문 1건이 15행으로 불어난다.
- **컬렉션은 하나만** fetch한다. 나머지는 `@BatchSize`나 별도 조회로 푼다.
- 컬렉션 타입을 `Set`으로 바꾸면 예외는 피하지만 **행 폭증은 그대로**다.

### 페이징과 같이 쓰면 메모리에서 자른다

```java
@EntityGraph(attributePaths = "items")        // 컬렉션 fetch
Page<Order> findByStatus(String status, Pageable pageable);
```

- 컬렉션을 JOIN하면 행 수가 엔티티 수와 달라져 **DB에서 `LIMIT`을 걸 수 없다.** Hibernate가 **전체를 읽어 메모리에서 페이징**한다.
- 로그에 `HHH000104: firstResult/maxResults specified with collection fetch; applying in memory` 경고가 뜬다. **데이터가 많으면 그대로 OOM이다.**
- 해결은 ID만 먼저 페이징해 조회하고, 그 ID로 그래프를 가져오는 2단계 방식이다.

### 그 외

- 적용됐는지 확인하려면 **SQL 로그를 본다.** 코드만 봐서는 알 수 없다.

```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
logging:
  level:
    org.hibernate.SQL: DEBUG
```

## 다른 방법과 비교

| 방법 | 범위 | 특징 |
| --- | --- | --- |
| **`@EntityGraph`** | **쿼리 단위** | 선언적. 메서드 이름 규칙과 같이 쓸 수 있다 |
| JPQL `join fetch` | 쿼리 단위 | `@Query`로 직접 작성. 조건이 복잡할 때 유리 |
| `FetchType.EAGER` | **엔티티 전체** | 모든 조회에 영향. **쓰지 않는 것을 권한다** |
| `@BatchSize` | 엔티티/필드 | N+1을 N/배치크기+1로 줄임. **컬렉션 여러 개**에 유효 |

- 쿼리 메서드 이름으로 해결되는 조회라면 `@EntityGraph`가 가장 간결하다.
- 컬렉션이 여럿이라 한 번에 못 가져오면 `@BatchSize`를 섞는다.

```java
@BatchSize(size = 100)
@OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
private List<Coupon> coupons;
```

## 관련 문서

- [JPA(Java Persistence API)](jpa.md) — ORM, IoC, Bean 개념
- [JPA 임베디드 타입과 복합 키](jpa-embedded-type-and-composite-key.md)
- [ListCrudRepository](list-crud-repository.md) — 기본 CRUD 제공 인터페이스
- [캐시 · 생명주기 어노테이션](cache-and-lifecycle-annotations.md) — 기동 시 캐시 적재와 `LazyInitializationException`
