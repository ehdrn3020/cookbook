# JPA 임베디드 타입과 복합 키

> 작성일: 2026-10-17

- 참조 : [Hibernate ORM — Embeddable values](https://docs.hibernate.org/orm/6.6/userguide/html_single/Hibernate_User_Guide.html#embeddables)
- 참조 : [Hibernate ORM — Composite identifiers](https://docs.hibernate.org/orm/6.6/userguide/html_single/Hibernate_User_Guide.html#identifiers-composite)
- 참조 : [Jakarta Persistence — EmbeddedId](https://jakarta.ee/specifications/persistence/3.2/apidocs/jakarta.persistence/jakarta/persistence/EmbeddedId.html)

- 세 어노테이션은 **붙는 위치가 다르고 역할이 다르다.** 헷갈리면 "누가 정의하고 누가 쓰는가"로 나눠 보면 된다.
- `@Embeddable`이 **값 타입을 정의**하고, `@Embedded`와 `@EmbeddedId`가 그걸 **가져다 쓴다.**

## 세 어노테이션의 역할

| 어노테이션 | 붙는 위치 | 의미 |
| --- | --- | --- |
| `@Embeddable` | **클래스** 선언부 | "나는 묻어 들어가는 값 타입이다" |
| `@EmbeddedId` | 엔티티의 **필드** | "이 값 타입을 내 **PK**로 쓴다" |
| `@Embedded` | 엔티티의 **필드** | "이 값 타입을 **일반 컬럼들**로 쓴다" |

- 값 타입은 **자기 테이블을 갖지 않는다.** 쓰는 쪽 엔티티의 테이블에 컬럼으로 펼쳐진다.
- 그래서 `@Entity`가 아니다. 식별자가 없고, 혼자서는 조회·저장되지 않는다.

## @Embeddable — 값 타입 정의

```java
import jakarta.persistence.Embeddable;

@Embeddable
public class Address {

    private String zipcode;
    private String city;
    private String street;

    protected Address() {}              // JPA가 요구하는 기본 생성자

    public Address(String zipcode, String city, String street) {
        this.zipcode = zipcode;
        this.city = city;
        this.street = street;
    }
    // getter만 두고 setter는 두지 않는다 (아래 "불변으로 만든다" 참조)
}
```

- 왜 쓰는가 — `zipcode`, `city`, `street`를 엔티티에 **평평하게 늘어놓는 대신 하나의 개념으로 묶는다.** 여러 엔티티에서 재사용할 수 있고, 주소 관련 로직을 이 클래스에 넣을 수 있다.
- 테이블은 그대로다. **객체 구조만 바뀐다.**

## @Embedded — 일반 컬럼으로 묻기

```java
@Entity
@Table(name = "member")
public class Member {

    @Id @GeneratedValue
    private Long id;

    private String name;

    @Embedded
    private Address address;            // zipcode, city, street 컬럼으로 펼쳐진다
}
```

```sql
-- 실제 테이블
CREATE TABLE member (
    id       BIGINT PRIMARY KEY,
    name     VARCHAR(255),
    zipcode  VARCHAR(255),   -- ┐
    city     VARCHAR(255),   -- ├ Address가 펼쳐진 것
    street   VARCHAR(255)    -- ┘
);
```

- `@Embedded`는 **생략해도 동작한다.** 대상 클래스에 `@Embeddable`이 있으면 Hibernate가 알아서 인식한다. 다만 **명시하는 편이 읽기 좋다.**
- 필드가 `null`이면 **구성 컬럼이 전부 `null`**이 된다. 반대로 전부 null인 행을 읽으면 객체도 `null`이 된다.

### @AttributeOverride — 같은 값 타입을 두 번 쓸 때

```java
@Entity
public class Order {

    @Id @GeneratedValue
    private Long id;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "zipcode", column = @Column(name = "ship_zipcode")),
        @AttributeOverride(name = "city",    column = @Column(name = "ship_city")),
        @AttributeOverride(name = "street",  column = @Column(name = "ship_street"))
    })
    private Address shippingAddress;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "zipcode", column = @Column(name = "bill_zipcode")),
        @AttributeOverride(name = "city",    column = @Column(name = "bill_city")),
        @AttributeOverride(name = "street",  column = @Column(name = "bill_street"))
    })
    private Address billingAddress;
}
```

- 같은 값 타입을 두 번 넣으면 **컬럼명이 충돌한다.** 한쪽 이상을 `@AttributeOverride`로 바꿔야 한다.
- 안 하면 기동 시점에 중복 컬럼 에러가 난다. **런타임이 아니라 부팅할 때 잡히는 건 다행인 쪽이다.**

## @EmbeddedId — 복합 키

```java
@Embeddable
public class OrderItemId implements Serializable {

    private Long orderId;
    private int  lineNo;

    protected OrderItemId() {}

    public OrderItemId(Long orderId, int lineNo) {
        this.orderId = orderId;
        this.lineNo = lineNo;
    }

    @Override
    public boolean equals(Object o) {                   // ★ 필수
        if (this == o) return true;
        if (!(o instanceof OrderItemId that)) return false;
        return lineNo == that.lineNo && Objects.equals(orderId, that.orderId);
    }

    @Override
    public int hashCode() {                             // ★ 필수
        return Objects.hash(orderId, lineNo);
    }
}
```

```java
@Entity
@Table(name = "order_item")
public class OrderItem {

    @EmbeddedId
    private OrderItemId id;             // PK가 (order_id, line_no) 두 컬럼

    private String productName;
    private int quantity;
}
```

### equals / hashCode는 선택이 아니다

- **Jakarta Persistence 명세가 요구한다.** 값 동등성이 DB의 기본 키와 맞아떨어져야 하기 때문이다.
- 안 만들면 **조용히 깨진다.** 영속성 컨텍스트는 1차 캐시를 `Map`처럼 쓰는데, `hashCode`가 기본 구현(객체 주소 기반)이면 같은 PK인데도 다른 키로 인식한다.
  - 같은 행을 두 번 조회하면 **다른 인스턴스**가 나온다
  - `findById`로 찾은 걸 `merge`했는데 **UPDATE가 아니라 INSERT**가 나간다
- `Serializable` 구현도 같이 해둔다. 명세상 요구되는 경우가 있고 비용이 없다.

### 조회할 때

```java
OrderItemId id = new OrderItemId(1001L, 1);
Optional<OrderItem> item = repository.findById(id);     // ID 타입이 OrderItemId
```

## @IdClass와의 차이

복합 키를 매핑하는 방식은 두 가지다.

```java
// ── @IdClass 방식 ──────────────────────────────
public class OrderItemId implements Serializable {      // @Embeddable 아님
    private Long orderId;
    private int  lineNo;
    // equals / hashCode 필요
}

@Entity
@IdClass(OrderItemId.class)
public class OrderItem {
    @Id private Long orderId;       // 엔티티에 필드를 직접 둔다
    @Id private int  lineNo;
}
```

| | `@EmbeddedId` | `@IdClass` |
| --- | --- | --- |
| 키 클래스 | `@Embeddable` 필요 | 평범한 클래스 |
| 엔티티의 필드 | **키 객체 하나** | **키 컬럼을 각각** |
| 접근 | `item.getId().getOrderId()` | `item.getOrderId()` |
| JPQL | `where o.id.orderId = :v` | `where o.orderId = :v` |
| 성격 | 객체지향적 — 키가 하나의 개념 | 관계형에 가까움 — 평평함 |

- **`@EmbeddedId`를 기본으로 권한다.** 키가 하나의 값 객체로 묶여 전달·비교가 쉽다.
- `@IdClass`는 JPQL이 짧아지는 장점이 있다. 기존 스키마에 맞춰 붙이는 경우 선택하기도 한다.

## Repository에서 쓰기

```java
public interface OrderItemRepository
        extends JpaRepository<OrderItem, OrderItemId> {
    //                       ↑엔티티     ↑@EmbeddedId 타입

    // 복합 키의 일부로 조회 — id를 거쳐 경로로 쓴다
    List<OrderItem> findByIdOrderId(Long orderId);

    // 필드명이 헷갈리면 @Query로 명시
    @Query("select i from OrderItem i where i.id.orderId = :orderId")
    List<OrderItem> findByOrder(@Param("orderId") Long orderId);

    // @Embedded 필드도 같은 방식
    // List<Member> findByAddressCity(String city);
}
```

- **메서드 이름이 `findByIdOrderId`처럼 길어진다.** `Id` → `OrderId` 경로를 그대로 쓰기 때문이다. 읽기 어려우면 `@Query`로 쓰는 편이 낫다.

## 주의할 것

### 값 타입은 불변으로 만든다

```java
Address addr = member1.getAddress();
addr.setCity("부산");               // ❌ member2도 같은 인스턴스를 참조하고 있으면 같이 바뀐다
```

- 값 타입 인스턴스를 **여러 엔티티가 공유하면 한쪽 수정이 다른 쪽에 번진다.** 부작용 추적이 매우 어렵다.
- **setter를 두지 않고**, 바꿀 때는 새 인스턴스로 교체한다.

```java
member.changeAddress(new Address("48058", "부산", "해운대로 1"));   // ✅
```

### 그 외

- `@Embeddable` 클래스에는 **기본 생성자가 필요하다.** `protected`로 두면 외부에서 쓰이지 않으면서 JPA는 쓸 수 있다.
- 값 타입 안에 다른 값 타입을 넣을 수 있다. 다만 깊어지면 `@AttributeOverride` 경로도 같이 깊어진다.
- **복합 키 엔티티는 키 생성 전략(`@GeneratedValue`)을 쓸 수 없다.** 값을 직접 채워 넣어야 한다.
- 복합 키는 조인과 인덱스가 복잡해진다. **대리 키(단일 `Long` PK) + 유니크 제약**으로 푸는 쪽이 운영에서 단순한 경우가 많다. 기존 스키마가 이미 복합 키일 때 쓰는 것이 일반적이다.

## 관련 문서

- [JPA(Java Persistence API)](jpa.md) — ORM, IoC, Bean 개념
- [ListCrudRepository](list-crud-repository.md) — 기본 CRUD 제공 인터페이스
- [트랜잭션 · 스케줄 · 비동기 어노테이션](transaction-schedule-async-annotations.md) — `@Transactional`
