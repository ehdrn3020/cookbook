# @Builder 패턴

> 작성일: 2026-09-01

- Lombok의 Builder 패턴 지원 어노테이션
- 가독성이 좋고, 필요한 값만 선택해서 넣기 편하며 어떤 값이 들어가는지 확인가능한게 장점
- 대체제로 생성자로도 만들 수 있음 ( new User("Kim", 30, "kim@test.com"); )

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
