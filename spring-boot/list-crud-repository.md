# ListCrudRepository

> 작성일: 2026-03-22

- 예제 코드

```javascript
public interface PersonRepository extends ListCrudRepository<Person, Long> {
    // 기본 CRUD는 상속으로 자동 제공
    // 필요하면 여기서 findByName 같은 메서드 추가 가능
}
>>>
// 저장
Person p = new Person("Alice");
repo.save(p);

// 조회
Optional<Person> findById(Long id);
List<Person> findAll();
```

- 기본적으로 CRUD 메서드를 제공하며 List로 반환 함
- Spring Data JPA에서는 구현 클래스를 직접 만들지 않아도, Spring이 실행 시점에 자동으로 Bean 등록
