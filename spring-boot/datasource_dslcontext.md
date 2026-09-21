# Datasource와 DSLContext
- 참조 : https://docs.spring.io/spring-boot/how-to/data-access.html
- 둘은 org.jooq.Configuration을 통해 연결된다.
```aiignore
# build.gradle
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-jooq")
}    
```

### Data Source
- DataSource는 커넥션을 어디서 얻을지 정의하는 인터페이스이다.
- Spring Boot는 `spring.datasource.*`를 읽어 DataSource를 자동 생성하며, 기본 구현체는 HikariCP다.
- 커넥션 풀 자체의 세부 옵션은 `spring.datasource.hikari.*`로 따로 덧씌운다.

### DSLContext
- DataSource 커넥션으로 SQL 어떻게 만들고 실행할지 담당한다.
- DSLContext는 jOOQ의 핵심 인터페이스로, SQL 쿼리를 작성하고 실행하는 데 사용된다.


### 조립순서
```aiignore
application.yml
   ↓  (Boot 바인딩)
DataSourceProperties          spring.datasource.*
   ↓  initializeDataSourceBuilder()
DataSource (HikariDataSource) ← spring.datasource.hikari.* 덧씌움
   ↓  (jOOQ 자동 설정)
TransactionAwareDataSourceProxy → DataSourceConnectionProvider
   ↓
org.jooq.Configuration        + SQLDialect, Settings, TransactionProvider
   ↓
DSLContext                    ← 애플리케이션이 실제로 주입받아 쓰는 것
```

### 커넥션 풀 설정을 어디에 쓰는가
- 자동 설정 DataSource는 접속 정보와 풀 옵션의 네임스페이스가 나뉜다.
```yaml
spring:
  datasource:
    url: "jdbc:mysql://localhost/test"     # 접속 정보
    username: "dbuser"
    password: "dbpass"
    hikari:                                # 풀 옵션은 여기
      maximum-pool-size: 30
      minimum-idle: 5
      idle-timeout: 600000
```
- 커스텀 DataSource를 직접 등록하면 `spring.datasource.hikari.*`가 적용되지 않는다. 직접 잡은 prefix 아래에 풀 옵션을 둬야 한다.

### 커스텀 DataSource 등록
- `@ConfigurationProperties` + `DataSourceBuilder` 조합이 기본형이다.
```java
@Configuration(proxyBeanMethods = false)
public class MyDataSourceConfiguration {
    @Bean
    @ConfigurationProperties("app.datasource")
    public HikariDataSource dataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }
}
```
- `type()`을 명시하는 이유 : 구현체가 고정되어야 IDE 자동완성과 설정 메타데이터가 동작한다.
- **주의** : Hikari는 `url`이 아니라 `jdbc-url`을 받는다. 위 방식으로 등록하면 yml도 `jdbc-url`로 써야 한다.
```yaml
app:
  datasource:
    jdbc-url: "jdbc:mysql://localhost/test"
    username: "dbuser"
    password: "dbpass"
```

- `url`이라는 이름을 그대로 쓰고 싶으면 `DataSourceProperties`를 거친다. `initializeDataSourceBuilder()`가 `url` → `jdbc-url` 변환을 대신 해준다. (조립순서 2~3단계가 바로 이 경로다)
```java
@Bean
@Primary
@ConfigurationProperties("app.datasource")
public DataSourceProperties dataSourceProperties() {
    return new DataSourceProperties();
}

@Bean
@ConfigurationProperties("app.datasource.configuration")   // 풀 옵션은 하위 prefix로 분리
public HikariDataSource dataSource(DataSourceProperties properties) {
    return properties.initializeDataSourceBuilder()
        .type(HikariDataSource.class).build();
}
```
```yaml
app:
  datasource:
    url: "jdbc:mysql://localhost/test"     # jdbc-url 아님
    username: "dbuser"
    password: "dbpass"
    configuration:
      maximum-pool-size: 30
```

### DataSource 2개 이상 쓰기
- 두 번째 DataSource에는 **`@Bean(defaultCandidate = false)`** 를 붙인다.
	- 이게 없으면 DataSource 빈이 2개로 보여 자동 설정이 물러나(back off) 첫 번째까지 안 만들어진다.
	- `defaultCandidate = false`인 빈은 `@Qualifier`로 명시해 찾을 때만 후보가 된다.
```java
@Configuration(proxyBeanMethods = false)
public class MyAdditionalDataSourceConfiguration {
    @Qualifier("second")
    @Bean(defaultCandidate = false)
    @ConfigurationProperties("app.datasource")
    public HikariDataSource secondDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }
}
```
- 첫 번째는 `spring.datasource.*`(자동 설정)를 그대로 쓰고, 두 번째만 `app.datasource.*` 같은 별도 prefix를 잡는다.
- 주입할 때는 같은 `@Qualifier`를 건다.
```java
@Autowired
@Qualifier("second")
private DataSource secondDataSource;
```

### DataSource별 DSLContext 분리
- 자동 설정 DSLContext는 기본 DataSource 하나만 바라본다. 두 번째는 직접 만든다.
```java
@Configuration(proxyBeanMethods = false)
public class MyJooqConfiguration {
    @Bean
    public DSLContext dslContext(@Qualifier("primary") DataSource dataSource) {
        return DSL.using(dataSource, SQLDialect.MYSQL);
    }

    @Qualifier("second")
    @Bean
    public DSLContext secondDslContext(@Qualifier("second") DataSource dataSource) {
        return DSL.using(dataSource, SQLDialect.MYSQL);
    }
}
```
- `DSL.using(dataSource, dialect)`로만 만들면 자동 설정이 끼워주던 기능이 빠진다. 아래 둘은 빈으로 제공되니 재사용해서 org.jooq.Configuration에 붙인다.
	- `ExceptionTranslatorExecuteListener` : jOOQ 예외를 Spring의 DataAccessException으로 변환
	- `SpringTransactionProvider` : `@Transactional`과 트랜잭션을 공유
