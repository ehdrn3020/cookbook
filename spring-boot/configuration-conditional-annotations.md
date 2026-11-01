# @Configuration 조건 어노테이션

> 작성일: 2026-11-01

- 참조 : [Creating Your Own Auto-configuration](https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html) — Spring Boot
- 참조 : [Locating Auto-configuration Candidates](https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html#features.developing-auto-configuration.locating-auto-configuration-candidates) — Spring Boot
- 참조 : [ConditionalOnProperty (API)](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/autoconfigure/condition/ConditionalOnProperty.html) — Spring Boot
- 참조 : [ConditionalOnResource (API)](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/autoconfigure/condition/ConditionalOnResource.html) — Spring Boot

- `@Conditional` 계열은 **"이 설정 클래스 / 이 빈을 등록할지 말지"**를 조건으로 거는 어노테이션이다.
- 코드를 바꾸지 않고 **설정·클래스패스·환경에 따라 구성을 바꾸는 것**이 목적이다. Spring Boot 자동 설정 전체가 이 구조 위에 서 있다.
- 조건이 틀리면 빈이 **아예 만들어지지 않는다.** 에러가 아니라 "없는 것"이 되므로 조용하다.

## 목차

- [조건 어노테이션 한눈에 보기](#조건-어노테이션-한눈에-보기)
- [@ConditionalOnProperty](#conditionalonproperty)
  - [속성](#속성)
  - [havingValue 매칭표](#havingvalue-매칭표)
  - [matchIfMissing](#matchifmissing)
  - [실제로 쓰는 모양 — 컨슈머 on/off](#실제로-쓰는-모양--컨슈머-onoff)
  - [설정 클래스에도 쓴다](#설정-클래스에도-쓴다)
- [클래스 조건 — @ConditionalOnClass](#클래스-조건--conditionalonclass)
  - [@Bean 메서드에는 그대로 못 쓴다](#bean-메서드에는-그대로-못-쓴다)
- [빈 조건 — @ConditionalOnBean / @ConditionalOnMissingBean](#빈-조건--conditionalonbean--conditionalonmissingbean)
  - [순서에 민감하다 — 자동 설정에만 쓸 것](#순서에-민감하다--자동-설정에만-쓸-것)
- [리소스 · 웹 조건](#리소스--웹-조건)
- [SpEL 조건 — @ConditionalOnExpression](#spel-조건--conditionalonexpression)
- [조건은 언제 평가되나](#조건은-언제-평가되나)
- [자동 설정은 어떻게 발견되나](#자동-설정은-어떻게-발견되나)
  - [AutoConfiguration.imports](#autoconfigurationimports)
  - [@AutoConfiguration과 순서](#autoconfiguration과-순서)
  - [클래스를 옮길 때 — .replacements](#클래스를-옮길-때--replacements)
- [주의할 것](#주의할-것)
  - [① 조건이 틀려도 에러가 안 난다](#-조건이-틀려도-에러가-안-난다)
  - [② 컬렉션 property에는 믿고 쓰면 안 된다](#-컬렉션-property에는-믿고-쓰면-안-된다)
  - [③ 클래스에 붙여도 클래스 생성 자체는 막지 않는다](#-클래스에-붙여도-클래스-생성-자체는-막지-않는다)
  - [④ property 조건과 빈 조건은 성격이 다르다](#-property-조건과-빈-조건은-성격이-다르다)
  - [⑤ 문자열 비교다](#-문자열-비교다)
  - [⑥ @Profile과 섞어 쓰지 말 것](#-profile과-섞어-쓰지-말-것)
- [관련 문서](#관련-문서)

## 조건 어노테이션 한눈에 보기

| 분류 | 어노테이션 | 무엇을 보는가 |
| --- | --- | --- |
| property | `@ConditionalOnProperty` | `Environment`의 property 값 |
| property | `@ConditionalOnBooleanProperty` | boolean property 전용 |
| 클래스 | `@ConditionalOnClass` / `@ConditionalOnMissingClass` | 클래스패스에 클래스가 있는지 |
| 빈 | `@ConditionalOnBean` / `@ConditionalOnMissingBean` | 지금까지 등록된 빈 정의 |
| 리소스 | `@ConditionalOnResource` | 지정한 리소스 경로의 존재 여부 |
| 웹 | `@ConditionalOnWebApplication` / `@ConditionalOnNotWebApplication` | 애플리케이션 타입 (servlet / reactive) |
| 웹 | `@ConditionalOnWarDeployment` / `@ConditionalOnNotWarDeployment` | 전통적 WAR 배포인지 |
| SpEL | `@ConditionalOnExpression` | SpEL 식의 결과 |
| 프로파일 | `@Profile` | 활성 프로파일 이름 (Spring Framework 쪽) |

공식 문서는 자동 설정 클래스에 **최소 하나의 `@Conditional`은 거의 항상 붙이라**고 권한다.

## @ConditionalOnProperty

`application.yml`에 설정한 property 값에 따라 빈 등록 여부를 결정한다. 애플리케이션 코드에서 가장 많이 쓰는 조건이다.

`Environment`에 올라온 property를 본다. yml이든 환경변수든 실행 인자든, **어디서 왔는지는 보지 않는다.** 이미 합쳐진 최종 값만 본다.

```java
@Component
@ConditionalOnProperty(prefix = "achievement.consumer.user", name = "enabled", havingValue = "true")
public class UserAttendConsumer {
    // achievement.consumer.user.enabled = true 일 때만 빈으로 등록된다
}
```

```yaml
achievement:
  consumer:
    user:
      enabled: true     # ← 이 값이 판단 기준
```

### 속성

| 속성 | 타입 | 기본값 | 의미 |
| --- | --- | --- | --- |
| `name` | `String[]` | `{}` | 검사할 property 이름. **여러 개면 전부 통과해야** 매칭된다 |
| `value` | `String[]` | `{}` | `name`의 별칭. 둘은 같은 것 |
| `prefix` | `String` | `""` | 각 `name` 앞에 붙는다. 점(`.`)은 자동으로 붙여준다 |
| `havingValue` | `String` | `""` | 기대하는 값. **비워두면 "`false`만 아니면 통과"** |
| `matchIfMissing` | `boolean` | `false` | property가 **아예 없을 때** 매칭으로 볼지 |

### havingValue 매칭표

공식 API 문서의 표를 그대로 옮긴 것이다. (✔ = 빈 등록됨)

| 실제 property 값 | `havingValue=""` | `havingValue="true"` | `havingValue="false"` | `havingValue="foo"` |
| --- | --- | --- | --- | --- |
| `"true"` | ✔ | ✔ | ✘ | ✘ |
| `"false"` | ✘ | ✘ | ✔ | ✘ |
| `"foo"` | ✔ | ✘ | ✘ | ✔ |

두 줄로 요약하면,

- **`havingValue`를 안 쓰면** → 값이 `false`가 아니기만 하면 통과한다. `"foo"`도 통과한다
- **`havingValue`를 쓰면** → 그 값과 **문자열이 똑같을 때만** 통과한다

### matchIfMissing

**property가 `Environment`에 아예 없을 때만** 참조되는 속성이다. 값이 있으면 이 속성은 무시된다.

```java
// 설정을 안 적으면 꺼짐 (기본값 false) — "명시적으로 켜야 도는 기능"
@ConditionalOnProperty(prefix = "achievement.consumer.event", name = "enabled", havingValue = "true")

// 설정을 안 적으면 켜짐 — "기본으로 도는 기능, 끄고 싶을 때만 설정"
@ConditionalOnProperty(prefix = "achievement.consumer.user", name = "enabled",
                       havingValue = "true", matchIfMissing = true)
```

기본으로 켜고 싶은 기능이면 `matchIfMissing = true`를 꼭 같이 써야 한다. 안 쓰면 **설정 파일에 키를 빠뜨린 순간 조용히 꺼진다.**

### 실제로 쓰는 모양 — 컨슈머 on/off

같은 jar를 띄우되 **환경마다 어떤 컨슈머가 돌지를 yml로 가른다.**

```java
@Slf4j
@Component
@RequiredArgsConstructor
@ConditionalOnProperty(
        prefix = "achievement.consumer.user",
        name = "enabled",
        havingValue = "true",
        matchIfMissing = true
)
public class UserAttendConsumer { ... }
```

```yaml
# application.yml (공통)
achievement:
  consumer:
    event:
      enabled: ${ACHIEVEMENT_CONSUMER_EVENT_ENABLED:false}
    user:
      enabled: ${ACHIEVEMENT_CONSUMER_USER_ENABLED:true}
    streamer:
      enabled: ${ACHIEVEMENT_CONSUMER_STREAMER_ENABLED:false}
```

```yaml
# application-local.yml — 로컬에서는 event만 켠다
achievement:
  consumer:
    event:
      enabled: ${ACHIEVEMENT_CONSUMER_EVENT_ENABLED:true}
    user:
      enabled: ${ACHIEVEMENT_CONSUMER_USER_ENABLED:false}
```

`${VAR:기본값}` 형태라 **배포 시에는 환경변수로 덮고, 안 주면 yml의 기본값이 쓰인다.** 두 겹 스위치인 셈이다. 어떤 yml이 읽히는지는 [환경별 배포 — 프로파일과 설정 파일](profile-based-deployment.md)을 본다.

### 설정 클래스에도 쓴다

```java
@Configuration
@ConditionalOnProperty(prefix = "spring.datasource-mydb", name = "jdbc-url")
public class DataSourceConfig { ... }
```

`havingValue`가 없으므로 **`jdbc-url`이 설정돼 있기만 하면** 통과한다. "값을 안 적었으면 이 DataSource는 안 쓴다"는 뜻으로, 이름 그대로 존재 여부만 보는 쓰임이다.

## 클래스 조건 — @ConditionalOnClass

클래스패스에 특정 클래스가 있을 때만 적용한다. **라이브러리가 의존성에 들어왔을 때만 설정을 켜는** 용도다.

```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
public class MyDataSourceAutoConfiguration { ... }
```

어노테이션 메타데이터를 **ASM으로 읽기 때문에**, `value`에 런타임 클래스패스에 없는 클래스를 적어도 된다. 문자열로 적으려면 `name`을 쓴다.

### @Bean 메서드에는 그대로 못 쓴다

JVM이 **조건을 검사하기 전에 반환 타입을 먼저 로드**하기 때문이다. 공식 문서는 중첩 설정 클래스로 격리하라고 안내한다.

```java
@AutoConfiguration
public final class MyAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnClass(SomeService.class)
    static class SomeServiceConfiguration {

        @Bean
        @ConditionalOnMissingBean
        SomeService someService() {
            return new SomeService();
        }
    }
}
```

합성 어노테이션(custom composed annotation) 안에서 쓸 때는 **반드시 `name`**을 써야 한다.

## 빈 조건 — @ConditionalOnBean / @ConditionalOnMissingBean

다른 빈이 있는지/없는지로 판단한다. `value`는 타입, `name`은 빈 이름, `search`는 컨텍스트 계층 탐색 범위를 정한다.

```java
@Bean
@ConditionalOnMissingBean          // 사용자가 직접 선언했으면 물러난다
SomeService someService() {
    return new SomeService();
}
```

`@Bean` 메서드에 붙이면 **대상 타입이 메서드 반환 타입으로 정해진다.** 조건 평가가 메서드 시그니처만 보기 때문에, **반환 타입을 가능한 한 구체적으로** 적어야 한다.

### 순서에 민감하다 — 자동 설정에만 쓸 것

공식 문서가 직접 경고하는 부분이다.

> 빈 조건은 "지금까지 처리된 것을 기준으로" 평가되므로, **빈 정의가 추가되는 순서에 매우 주의해야 한다.**

그래서 `@ConditionalOnBean` / `@ConditionalOnMissingBean`은 **자동 설정 클래스에만** 쓰라고 권한다. 자동 설정은 사용자 정의 빈이 모두 등록된 **뒤에** 로드되는 것이 보장되기 때문이다.

일반 `@Component`에 붙이면 컴포넌트 스캔 순서에 따라 결과가 달라질 수 있다. 애플리케이션 코드에서는 `@ConditionalOnProperty`를 쓰는 편이 안전하다.

## 리소스 · 웹 조건

```java
@ConditionalOnResource(resources = "classpath:/config/feature.properties")
public class FeatureConfiguration { ... }
```

`resources`는 `String[]`이고, Spring 표준 리소스 문법(`classpath:`, `file:`)을 쓴다.

```java
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
public class MyWebConfiguration { ... }
```

| 어노테이션 | 매칭 조건 |
| --- | --- |
| `@ConditionalOnWebApplication(type = SERVLET)` | `WebApplicationContext`를 쓰거나, `session` 스코프가 있거나, `ConfigurableWebEnvironment`가 있을 때 |
| `@ConditionalOnWebApplication(type = REACTIVE)` | `ReactiveWebApplicationContext` 또는 `ConfigurableReactiveWebEnvironment`일 때 |
| `@ConditionalOnWarDeployment` | **전통적 WAR 배포일 때만.** 내장 웹 서버에는 매칭되지 않는다 |

컨슈머·배치처럼 웹이 아닌 애플리케이션에서는 `@ConditionalOnNotWebApplication` 쪽이 쓸모 있다.

## SpEL 조건 — @ConditionalOnExpression

```java
@ConditionalOnExpression("${achievement.consumer.user.enabled:true} and ${achievement.season.id:0} > 0")
public class SomeConfiguration { ... }
```

여러 property를 조합해야 할 때만 쓴다. 공식 문서의 경고가 있다.

> 식 안에서 빈을 참조하면 **그 빈이 아주 이른 시점에 초기화된다.** configuration properties 바인딩 같은 후처리 대상에서 빠지므로, 상태가 불완전할 수 있다.

property만 참조하는 선에서 쓰고, **빈 참조는 피하는 게 안전하다.**

## 조건은 언제 평가되나

조건이 평가될 시점에는 **모든 property가 이미 확정되어 있어야 한다.**

```
SpringApplication.run()
  │
  ├─ prepareEnvironment()        ← ① 프로파일 결정, application-{profile}.yml 로드
  │                                 (자세한 흐름은 "환경별 배포" 문서)
  │
  └─ refreshContext()            ← ② 조건이 평가되는 단계
      └ ConfigurationClassPostProcessor
          └ ConditionEvaluator
              └ OnPropertyCondition  ★ 여기서 @ConditionalOnProperty 판정
                                       ①에서 확정된 Environment를 읽기만 한다
```

순서가 ①→② 로 고정돼 있다는 점이 중요하다.

- **프로파일이 먼저 정해지고 → yml이 로드되고 → 그다음 조건이 평가된다**
- 그래서 `@ConditionalOnProperty`는 **프로파일에 따라 결과가 달라진다.** 같은 코드, 같은 jar인데 `SPRING_PROFILES_ACTIVE=local`이면 켜지고 `prod`면 꺼지는 이유다
- 반대로, 런타임에 property를 바꿔도 **이미 등록된 빈은 사라지지 않는다.** 기동 시점에 한 번 판정하고 끝이다

> ②의 `OnPropertyCondition`은 내부 클래스라 공식 레퍼런스에 이름으로 명시돼 있지는 않다. 공식 문서가 보장하는 것은 **"`Environment` property를 검사한다"**와 **"조건은 지금까지 처리된 것을 기준으로 평가된다"**까지다.

## 자동 설정은 어떻게 발견되나

직접 만든 설정 클래스는 컴포넌트 스캔으로 잡히지만, **자동 설정 클래스는 스캔으로 잡히면 안 된다.** 발견 경로가 따로 있다.

### AutoConfiguration.imports

jar 안의 **이 경로**에 한 줄에 하나씩 FQCN을 적는다.

```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

```
com.mycorp.libx.autoconfigure.LibXAutoConfiguration
# 주석은 # 으로 시작한다
com.example.Outer$NestedAutoConfiguration
```

- 중첩 클래스는 `.`이 아니라 **`$`**로 적는다
- 자동 설정은 **오직 이 파일에 이름이 올라야만** 로드된다
- 별도 패키지에 두고 **컴포넌트 스캔 대상에 절대 넣지 않는다**
- 자동 설정 클래스 안에서 컴포넌트 스캔을 켜지 말고, **`@Import`를 쓴다**

Spring 자체 자동 설정 목록을 보려면 `spring-boot-autoconfigure` jar의 같은 파일을 열어 보면 된다.

### @AutoConfiguration과 순서

```java
@AutoConfiguration(after = WebMvcAutoConfiguration.class)
public class MyWebAutoConfiguration { ... }
```

`@AutoConfiguration`은 **`@Configuration`이 메타 어노테이션으로 붙어 있어** 그 자체가 설정 클래스다.

| 지정 방법 | 쓰임 |
| --- | --- |
| `@AutoConfiguration(before / beforeName / after / afterName)` | 특정 자동 설정의 앞/뒤를 지정 |
| `@AutoConfigureBefore` / `@AutoConfigureAfter` | 위와 같은 일을 하는 전용 어노테이션 |
| `@AutoConfigureOrder` | `@Order`와 같은 의미. **서로를 모르는** 클래스끼리 순서를 줄 때 |

중요한 단서가 있다.

> 이 순서는 **빈 정의가 등록되는 순서**에만 영향을 준다. **빈이 생성되는 순서**는 의존 관계와 `@DependsOn`이 결정한다.

`@ConditionalOnBean` 계열이 이 순서에 민감한 이유가 여기 있다.

### 클래스를 옮길 때 — .replacements

자동 설정 클래스를 이동·이름 변경하면 기존 참조가 깨진다. 이 파일로 매핑을 남긴다.

```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.replacements
```

```
old.Class=new.Class
```

그다음 `.imports` 파일은 **새 클래스만** 가리키도록 고친다.

## 주의할 것

### ① 조건이 틀려도 에러가 안 난다

빈이 없는 것뿐이라 기동은 성공한다. **"컨슈머를 띄웠는데 메시지를 안 먹는다"**가 전형적인 증상이다. `--debug`로 띄우면 조건 평가 리포트가 콘솔에 찍힌다.

```bash
java -jar app.jar --debug
```

```
Negative matches:
-----------------
   UserAttendConsumer:
      Did not match:
         - @ConditionalOnProperty (achievement.consumer.user.enabled=true) did not find property 'enabled'
```

Actuator를 쓰면 `/actuator/conditions`로도 볼 수 있다.

### ② 컬렉션 property에는 믿고 쓰면 안 된다

공식 API 문서가 직접 경고한다. `name = "example.values"`는 `spring.example.values`가 있으면 매칭되지만, **`spring.example.values[0]`만 있으면 매칭되지 않는다.** 리스트를 조건으로 삼아야 하면 `Condition`을 직접 구현하라고 안내한다.

### ③ 클래스에 붙여도 클래스 생성 자체는 막지 않는다

문서 표현으로는 "조건이 실패하면 **빈으로 등록되는 것만** 막힌다". `@Configuration` 클래스가 아예 로드조차 안 된다고 생각하면 안 된다.

### ④ property 조건과 빈 조건은 성격이 다르다

| | 보는 대상 | 평가 순서 영향 |
| --- | --- | --- |
| `@ConditionalOnProperty` | `Environment` — 기동 초기에 이미 확정 | **없음** |
| `@ConditionalOnBean` / `@ConditionalOnMissingBean` | 지금까지 등록된 빈 정의 | **큼. 순서에 따라 결과가 달라진다** |

### ⑤ 문자열 비교다

`havingValue = "true"`는 **문자열 `"true"`와 비교**한다. yml에서 `enabled: TRUE`로 적으면 안 맞을 수 있다. boolean 전용으로 `@ConditionalOnBooleanProperty`가 따로 있다.

### ⑥ @Profile과 섞어 쓰지 말 것

프로파일 이름으로만 가르면 되는 경우엔 `@Profile("local")`이 더 단순하다. **기능 단위로 켜고 끄려면** `@ConditionalOnProperty`가 맞다. 프로파일을 늘리지 않고도 스위치를 추가할 수 있기 때문이다.

둘을 같은 클래스에 겹쳐 붙이면 **어느 쪽 때문에 안 떴는지 추적하기 어려워진다.** 하나로 정해서 쓴다.

## 관련 문서

- [환경별 배포 — 프로파일과 설정 파일](profile-based-deployment.md)
- [@Bean](bean.md)
- [프록시와 self-invocation](proxy-and-self-invocation.md)
- [Datasource와 DSLContext](datasource_dslcontext.md)
