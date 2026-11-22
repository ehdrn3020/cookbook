# @Builder 패턴

> 작성일: 2026-09-01

- 참조 : [@Builder](https://projectlombok.org/features/Builder) — Project Lombok

- Lombok의 Builder 패턴 지원 어노테이션
- 가독성이 좋고, 필요한 값만 선택해서 넣기 편하며 어떤 값이 들어가는지 확인가능한게 장점
- 대체제로 생성자로도 만들 수 있음 ( new User("Kim", 30, "kim@test.com"); )

## 목차

- [기본 사용법](#기본-사용법)
- [@Builder.Default](#builderdefault)
  - [붙이지 않았을 때](#붙이지-않았을-때)
  - [붙였을 때](#붙였을-때)
  - [동작 방식 — 초기값이 static 메서드로 옮겨진다](#동작-방식--초기값이-static-메서드로-옮겨진다)
  - [생성자와 같이 쓸 때](#생성자와-같이-쓸-때)
  - [컬렉션은 @Singular도 고려](#컬렉션은-singular도-고려)
- [관련 문서](#관련-문서)

## 기본 사용법

```java
@Builder
public class User {
    private String name;
    private int age;
    private String email;
}
.....
User user = User.builder()
        .name("Kim")
        .age(30)
        .email("kim@test.com")
        .build();
```

## @Builder.Default

- 필드에 초기값을 써 두어도 **builder로 만들면 그 초기값은 무시**되고 0 / null / false가 들어간다
- 초기값을 builder의 기본값으로 쓰고 싶으면 필드에 `@Builder.Default`를 붙인다
- 클래스에 `@Builder`를 붙였을 때만 쓸 수 있다 (메서드·생성자에 붙인 경우는 해당 없음)

### 붙이지 않았을 때

```java
@Getter
@Builder
public class User {
    private String name;
    private int age;
    private String role = "USER";                    // builder에서는 무시됨
    private List<String> tags = new ArrayList<>();   // builder에서는 무시됨
}
.....
User user = User.builder()
        .name("Kim")
        .build();

user.getRole();          // null  ← "USER"가 아니다
user.getTags().add("a"); // NullPointerException
```

- 컴파일할 때 Lombok이 경고를 띄운다

```
@Builder will ignore the initializing expression entirely. If you want the initializing
expression to serve as default, add @Builder.Default. If it is not supposed to be settable
during building, make the field final.
```

### 붙였을 때

```java
@Getter
@Builder
public class User {
    private String name;
    private int age;

    @Builder.Default
    private String role = "USER";

    @Builder.Default
    private List<String> tags = new ArrayList<>();

    @Builder.Default
    private long createdAt = System.currentTimeMillis();
}
.....
User user1 = User.builder()
        .name("Kim")
        .build();
user1.getRole();   // "USER"
user1.getTags();   // []  (빈 리스트, NPE 안 남)

User user2 = User.builder()
        .name("Lee")
        .role("ADMIN")   // 직접 넣으면 기본값 대신 이 값이 들어간다
        .build();
user2.getRole();   // "ADMIN"
```

### 동작 방식 — 초기값이 static 메서드로 옮겨진다

- 필드의 초기값은 **제거되고 static 메서드로 옮겨진다**
- builder는 값이 set 되었는지 플래그로 기억했다가, 안 됐으면 `build()` 시점에 그 static 메서드를 호출한다

```java
// Lombok이 만드는 코드 (개략)
public class User {
    private String role;                                   // 초기값 제거됨

    private static String $default$role() { return "USER"; }

    public static class UserBuilder {
        private String role$value;
        private boolean role$set;

        public UserBuilder role(String role) {
            this.role$value = role;
            this.role$set = true;
            return this;
        }

        public User build() {
            String role$value = this.role$value;
            if (!this.role$set) role$value = User.$default$role();
            return new User(name, age, role$value, ...);
        }
    }
}
```

- `$default$...`, `...$value`, `...$set`은 구현 세부사항이다. 코드에서 직접 쓰면 안 된다
- 여기서 나오는 결과 두 가지
  - `build()` 할 때마다 초기값 식이 새로 실행된다 → `new ArrayList<>()`가 객체마다 따로 생기고, `createdAt`은 build 시점 시각이 된다
  - static 메서드로 옮겨지므로 **초기값에서 `this`, `super`, static이 아닌 필드·메서드를 참조할 수 없다**

### 생성자와 같이 쓸 때

- Lombok이 만든 생성자(`@NoArgsConstructor` 등)는 기본값을 쓴다 (Lombok 1.18.2부터)
- **직접 작성한 생성자는 기본값을 쓰지 않는다.** 초기값이 필드에서 제거됐기 때문이다

```java
@Getter
@Builder
@NoArgsConstructor(access = AccessLevel.PROTECTED)   // JPA 엔티티에서 흔한 조합
@AllArgsConstructor
public class User {
    private String name;

    @Builder.Default
    private String role = "USER";

    public User(String name) {     // 직접 작성한 생성자
        this.name = name;
    }
}
.....
User.builder().name("Kim").build().getRole();   // "USER"
new User("Kim").getRole();                      // null  ← 직접 넣어줘야 한다
```

### 컬렉션은 @Singular도 고려

- 컬렉션 필드에 `@Singular`를 붙이면 원소를 하나씩 넣는 메서드가 생긴다
- 아무것도 안 넣고 `build()` 하면 빈 컬렉션이 되고, 결과는 **불변 컬렉션**이다

```java
@Builder
public class User {
    private String name;
    @Singular
    private List<String> tags;
}
.....
User.builder()
        .name("Kim")
        .tag("a")          // 하나씩 추가 (이름은 tags → tag 로 자동 단수화)
        .tag("b")
        .build();          // tags = [a, b], 수정 불가
```

- 나중에 원소를 추가해야 하면 `@Builder.Default` + `new ArrayList<>()`, 만든 뒤 바뀌면 안 되면 `@Singular`

## 관련 문서

- [Lombok](lombok.md)
