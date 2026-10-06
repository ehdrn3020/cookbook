# @Bean

> 작성일: 2026-01-06

- 참조 : [Using the @Bean Annotation](https://docs.spring.io/spring-framework/reference/core/beans/java/bean-annotation.html)
- 참조 : [The IoC Container](https://docs.spring.io/spring-framework/reference/core/beans.html)

- @Bean은 프레임워크가 제공하는 기본 동작 대신 사용자가 정의한 구현체를 주입하기 위한 확장 포인트
- @Autowired 어노테이션을 사용해, Bean에 등록된 객체를 사용
- @Bean은 의존성 주입의 기준은 함수명이 아닌 "타입" ( 같은 타임 여러 함수명이면 우선순위로 구분 )

## 기본 형태

- `@Configuration` 클래스 안에서 메서드에 붙이고, **반환된 객체가 컨테이너에 등록**된다.
- **메서드명이 곧 빈 이름**이 된다. (위의 "함수명이 아닌 타입" 기준과 헷갈리기 쉬운데, 이름은 붙되 주입 기준이 타입이라는 뜻)

```java
@Configuration
public class AppConfig {

    @Bean                                  // 빈 이름 = "paymentClient"
    public PaymentClient paymentClient() {
        return new PaymentClient("https://api.example.com", 3000);
    }

    @Bean                                  // 다른 빈을 파라미터로 받으면 주입된다
    public OrderService orderService(PaymentClient paymentClient) {
        return new OrderService(paymentClient);
    }
}
```

## @Component와 언제 갈리나

| | @Component (@Service, @Repository) | @Bean |
| --- | --- | --- |
| 붙이는 대상 | 클래스 | `@Configuration` 안의 메서드 |
| 등록 방식 | 컴포넌트 스캔이 찾아서 자동 등록 | 개발자가 직접 생성해 반환 |
| 쓰는 경우 | **내가 작성한 클래스** | **외부 라이브러리 클래스**, 생성에 로직이 필요한 경우 |

- 외부 라이브러리 클래스에는 `@Component`를 붙일 수 없다. 그래서 `KafkaTemplate`, `ObjectMapper`, `DataSource` 같은 것은 `@Bean`으로 등록한다.

## 같은 타입이 여러 개일 때

- 주입 기준이 타입이라, 같은 타입 빈이 2개면 어느 것을 넣을지 모호해져 실행이 실패한다. 해결은 두 가지다.

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary                               // ① 기본 후보로 지정
    public DataSource mainDataSource() { ... }

    @Bean
    @Qualifier("archive")                  // ② 이름표를 달아 지목해서 꺼내 쓰기
    public DataSource archiveDataSource() { ... }
}
```

```java
@Service
public class ReportService {

    private final DataSource main;
    private final DataSource archive;

    // 생성자 주입 권장 — 필드가 final이 되고, 테스트에서 수동 주입이 쉽다
    public ReportService(DataSource main,
                         @Qualifier("archive") DataSource archive) {
        this.main = main;        // @Primary가 붙은 mainDataSource
        this.archive = archive;  // @Qualifier로 지목한 쪽
    }
}
```

- 참고 : [Datasource와 DSLContext](datasource_dslcontext.md) 에서 DataSource를 2개 쓸 때 `@Bean(defaultCandidate = false)`까지 필요한 이유를 다룬다.

## 자동 설정을 덮어쓰는 지점

- Spring Boot 자동 설정 대부분은 **`@ConditionalOnMissingBean`** 으로 선언돼 있다. "그 타입의 빈이 없을 때만 내가 만든다"는 뜻이다.
- 그래서 같은 타입의 빈을 직접 `@Bean`으로 등록하면 **자동 설정이 물러나고(back off) 내 것이 쓰인다.** 이게 맨 위 "기본 동작 대신 사용자 구현체를 주입하는 확장 포인트"의 실제 동작이다.

```java
@Configuration
public class JacksonConfig {

    @Bean   // Boot가 만들어주던 ObjectMapper 대신 이게 쓰인다
    public ObjectMapper objectMapper() {
        return JsonMapper.builder()
                .addModule(new JavaTimeModule())
                .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
                .build();
    }
}
```

## 스코프와 생명주기

- 기본 스코프는 **싱글톤**이다. 메서드를 여러 번 호출해도 컨테이너 안에는 인스턴스가 하나다.
- 종료 시 정리가 필요한 자원은 `destroyMethod`를 지정한다. (반환 타입에 `close()`가 있으면 Spring이 알아서 호출하기도 한다)

```java
@Bean(initMethod = "start", destroyMethod = "shutdown")
public MetricsReporter metricsReporter() {
    return new MetricsReporter();
}
```

- `@Configuration(proxyBeanMethods = false)` : 설정 클래스 안에서 `@Bean` 메서드끼리 서로 호출하지 않는다면 꺼두는 편이 가볍다. 켜져 있으면(기본값) Spring이 CGLIB 프록시를 만들어 싱글톤을 보장한다.

### 싱글톤이 아닌 스코프

- `@Scope`로 바꾼다. 웹 환경이면 `request`, `session`도 쓸 수 있다.

```java
@Bean
@Scope("prototype")          // 주입될 때마다 새 인스턴스
public Parser parser() {
    return new Parser();
}
```

- **주의** : 싱글톤 빈이 수명이 짧은 빈(`request`, `prototype`)을 주입받으면, 처음 주입된 인스턴스가 계속 남아 스코프가 무의미해진다. 이때 `proxyMode`로 프록시를 끼워 **호출 시점마다 실제 인스턴스를 찾아가게** 한다.

```java
@Bean
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public UserPreferences userPreferences() {
    return new UserPreferences();
}
```

## 빈 이름 지정과 별칭

- 기본값은 메서드명이지만, 직접 지정할 수 있다.

```java
@Bean("myThing")             // 빈 이름 = "myThing" (메서드명 무시)
public Thing thing() {
    return new Thing();
}
```

- 배열로 주면 **첫 번째가 이름, 나머지는 별칭**이 된다. 같은 빈을 모듈마다 다른 이름으로 참조해야 할 때 쓴다.

```java
@Bean({"dataSource", "subsystemA-dataSource", "subsystemB-dataSource"})
public DataSource dataSource() {
    return new EmbeddedDatabaseBuilder().build();
}
```

- `@Description`은 빈에 설명을 붙인다. 액추에이터나 모니터링 도구에서 빈 목록을 볼 때 드러난다.

```java
@Bean
@Description("Provides a basic example of a bean")
public Thing thing() {
    return new Thing();
}
```

## 관련 문서

- [Spring for Kafka](kafka-template.md) — `@Bean`으로 등록해 쓰는 대표 사례
- [Datasource와 DSLContext](datasource_dslcontext.md) — 자동 설정 back off와 다중 빈 등록
- [JPA(Java Persistence API)](jpa.md) — @Autowired, IoC, Bean 개념
