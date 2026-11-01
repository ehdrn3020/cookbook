# 환경별 배포 — 프로파일과 설정 파일

> 작성일: 2026-11-01

- 참조 : [Profiles](https://docs.spring.io/spring-boot/reference/features/profiles.html) — Spring Boot
- 참조 : [Externalized Configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html) — Spring Boot
- 참조 : [Using Profiles with Compose](https://docs.docker.com/compose/how-tos/profiles/) — Docker

- **같은 jar 하나로 local / dev / stag / prod를 모두 띄우는 것**이 목적이다. 환경마다 빌드를 따로 하지 않는다.
- 환경 차이는 코드가 아니라 **프로파일 + 설정 파일 + 환경변수** 세 겹으로 흡수한다.
- 어떤 설정이 최종적으로 적용됐는지는 **PropertySource 우선순위**가 결정한다. 이것만 알면 대부분의 "왜 이 값이 안 먹지"가 풀린다.

## 목차

- [설정이 정해지는 순서](#설정이-정해지는-순서)
- [프로파일을 지정하는 방법](#프로파일을-지정하는-방법)
  - [방법 1 — 환경변수](#방법-1--환경변수)
  - [방법 2 — java 실행 인자](#방법-2--java-실행-인자)
    - [환경변수와 같이 주면 인자가 이긴다](#환경변수와-같이-주면-인자가-이긴다)
  - [방법 3 — docker compose --profile (Spring과 무관)](#방법-3--docker-compose---profile-spring과-무관)
- [설정 파일을 나누는 두 가지 방법](#설정-파일을-나누는-두-가지-방법)
  - [방식 A — 한 파일 안에서 --- 로 나누기](#방식-a--한-파일-안에서-----로-나누기)
  - [방식 B — 파일을 나누기](#방식-b--파일을-나누기)
  - [무엇이 다른가](#무엇이-다른가)
  - [섞어 쓰면 파일 분리가 이긴다](#섞어-쓰면-파일-분리가-이긴다)
  - [spring.profiles.active는 아무 데나 못 쓴다](#springprofilesactive는-아무-데나-못-쓴다)
  - [어느 쪽을 쓰나](#어느-쪽을-쓰나)
- [환경변수 이름 변환 규칙](#환경변수-이름-변환-규칙)
- [우선순위](#우선순위)
- [실제 배포 모양](#실제-배포-모양)
- [관련 문서](#관련-문서)

## 설정이 정해지는 순서

```
SpringApplication.run()
  │
  ├─ prepareEnvironment()                                   ← ① 환경이 확정되는 단계
  │   └ listeners.environmentPrepared(...)                     ApplicationEnvironmentPreparedEvent 발행
  │       └ EnvironmentPostProcessorApplicationListener
  │           └ ConfigDataEnvironmentPostProcessor.postProcessEnvironment()
  │               └ ConfigDataEnvironment.processAndApply()  ★ 프로파일을 읽고
  │                                                            application-{profile}.yml 을 로드
  │                                                            (local / dev / stag / prod)
  │
  └─ refreshContext()                                       ← ② 빈이 만들어지는 단계
      └ ConfigurationClassPostProcessor
          └ ConditionEvaluator                                 @ConditionalOnProperty 등이 여기서 판정
                                                               ①에서 확정된 Environment를 읽기만 한다
```

- **프로파일이 먼저 정해지고 → yml이 로드되고 → 그다음에야 빈이 만들어진다**
- 그래서 **런타임에 설정을 바꿔도 이미 등록된 빈은 바뀌지 않는다.** 기동 시점에 한 번 결정되고 끝이다
- 조건 어노테이션 쪽은 [@Configuration 조건 어노테이션](configuration-conditional-annotations.md)에 정리돼 있다

> ①의 호출 체인은 Spring Boot 소스 기준이다. 공식 레퍼런스에 이 클래스 이름들이 순서대로 나열돼 있지는 않다.

## 프로파일을 지정하는 방법

위 ①에서 "어떤 yml을 읽을지"를 결정하는 값이다. 혼동하기 쉬운 셋을 구분한다.

| | 무엇 |
| --- | --- |
| `SPRING_PROFILES_ACTIVE` | 환경변수. Spring 프로파일 지정 |
| `--spring.profiles.active=dev` | java 실행 인자. 같은 기능, **우선순위 더 높음** |
| `docker compose --profile <이름>` | **compose가 어떤 서비스를 띄울지** 고르는 기능 |

### 방법 1 — 환경변수

셸에서 export 하거나, 한 줄로 붙여서 준다.

```bash
export SPRING_PROFILES_ACTIVE=stag
java -jar achievement-consumer.jar
```

```bash
# 한 줄로
SPRING_PROFILES_ACTIVE=stag java -jar achievement-consumer.jar
```

컨테이너로 띄울 때.

```bash
docker run -e SPRING_PROFILES_ACTIVE=stag achievement-consumer:stag
```

compose에서는 **호스트 환경변수나 `.env`를 컨테이너로 넘기는** 모양이 된다.

```yaml
services:
  achievement-consumer-user:
    environment:
      SPRING_PROFILES_ACTIVE: ${SPRING_PROFILES_ACTIVE:-stag}
```

```bash
docker compose --env-file .env.prod up -d
```

`${SPRING_PROFILES_ACTIVE:-stag}`는 **`.env`나 호스트에 값이 없으면 `stag`**를 쓴다는 뜻이다. 환경 파일만 바꿔 끼우면 같은 compose 파일로 dev/stag/prod를 모두 띄울 수 있다.

Kubernetes라면 이렇게 된다.

```yaml
env:
  - name: SPRING_PROFILES_ACTIVE
    value: "prod"
```

어느 쪽이든 결과는 같다. `application.yml`을 먼저 읽고 **`application-{프로파일}.yml`을 덮어씌운다.**

### 방법 2 — java 실행 인자

```bash
java -jar achievement-consumer.jar --spring.profiles.active=dev
```

**인자는 jar 파일명 뒤에 와야 한다.** 앞에 쓰면 JVM 옵션으로 해석되어 실행이 실패한다.

```bash
java --spring.profiles.active=dev -jar achievement-consumer.jar   # ✗ Unrecognized option
```

Gradle로 바로 띄울 때는 `--args`로 넘긴다.

```bash
./gradlew bootRun --args='--spring.profiles.active=local'
```

시스템 프로퍼티(`-D`)로도 같은 일을 할 수 있다. 이건 **JVM 옵션이라 jar 앞**에 온다.

```bash
java -Dspring.profiles.active=dev -jar achievement-consumer.jar
```

여러 개를 켤 때는 콤마로 잇는다.

```bash
java -jar achievement-consumer.jar --spring.profiles.active=prod,metrics
```

#### 환경변수와 같이 주면 인자가 이긴다

```bash
SPRING_PROFILES_ACTIVE=stag java -jar achievement-consumer.jar --spring.profiles.active=dev
# → dev 로 뜬다
```

PropertySource 순서가 **환경변수 5번 < 시스템 프로퍼티 6번 < 커맨드라인 인자 11번**이기 때문이다. 배포는 환경변수로 고정해두고, **일회성으로 다른 프로파일을 띄워 볼 때** 인자로 덮는 식으로 쓴다.

### 방법 3 — docker compose --profile (Spring과 무관)

이름만 같고 **완전히 다른 것**이다. Spring 프로파일이 아니라, **compose가 어떤 서비스를 띄울지** 고르는 기능이다. 컨테이너 안의 Spring에는 아무 영향이 없다.

```yaml
services:
  achievement-consumer-user:
    # profiles 키 없음 → 항상 뜬다
    environment:
      SPRING_PROFILES_ACTIVE: stag

  achievement-consumer-streamer:
    profiles: ["streamer"]        # ← compose 프로파일
    environment:
      SPRING_PROFILES_ACTIVE: stag
```

```bash
docker compose up -d
# → achievement-consumer-user 만 뜬다

docker compose --profile streamer up -d
# → 둘 다 뜬다

COMPOSE_PROFILES=streamer docker compose up -d
# → 같은 결과. 환경변수로 주는 형태

docker compose --profile "*" up -d
# → 모든 프로파일을 켠다
```

공식 문서가 정한 규칙이다.

- `profiles:`가 **없는** 서비스는 항상 뜬다
- 여러 개는 `--profile a --profile b` 또는 `COMPOSE_PROFILES=a,b`
- 서비스 이름을 직접 지정하면(`docker compose up achievement-consumer-streamer`) 프로파일이 꺼져 있어도 뜬다

두 `profile`의 차이를 한 줄로 정리하면 이렇다.

| | 무엇을 고르는가 |
| --- | --- |
| `SPRING_PROFILES_ACTIVE` | 컨테이너 **안**에서 어떤 yml을 읽을지 |
| `docker compose --profile` | 컨테이너를 **띄울지 말지** |

## 설정 파일을 나누는 두 가지 방법

**기능은 같다.** 둘 다 `ConfigDataEnvironment.processAndApply()`가 처리하는 **같은 메커니즘의 두 가지 표기법**이고 결과도 같다. 다른 건 **적용 순서와 걸 수 있는 조건의 범위**다.

### 방식 A — 한 파일 안에서 --- 로 나누기

YAML 표준 멀티 도큐먼트 문법이다. 하이픈 세 개가 문서의 끝이자 다음 문서의 시작이다.

```yaml
achievement:
  consumer:
    user:
      enabled: true          # 공통

---
spring:
  config:
    activate:
      on-profile: "prod"     # ← 이 문서는 prod 일 때만 적용
achievement:
  consumer:
    user:
      enabled: false
```

`.properties`에서는 `#---` 또는 `!---` 주석이 구분자다. **앞에 공백이 없어야 하고 하이픈이 정확히 세 개**여야 하며, 구분자 바로 앞뒤 줄은 같은 주석 접두사면 안 된다.

### 방식 B — 파일을 나누기

```yaml
# application.yml
achievement:
  consumer:
    user:
      enabled: true
```

```yaml
# application-prod.yml
achievement:
  consumer:
    user:
      enabled: false
```

`prod`로 띄우면 **둘 다 `enabled: false`**가 된다. `@ConditionalOnProperty` 입장에선 구분이 안 된다. 최종 `Environment` 값만 보기 때문이다.

### 무엇이 다른가

| | 한 파일 안 `---` | 파일 분리 |
| --- | --- | --- |
| 적용 순서 | **위에서 아래로.** 뒤 문서가 이김 | **profile-specific이 항상 공통을 덮음** |
| 조건 | `on-profile` + `on-cloud-platform` | 프로파일 **이름만** |
| 프로파일 식 | `on-profile: "prod \| staging"` **가능** | 불가 (파일명 하나 = 프로파일 하나) |
| `spring.profiles.active` | **맨 위 공통 문서에만** | `application.yml`에만 |
| 배포 시 교체 | 파일 통째로 교체 | **해당 환경 파일만 교체 가능** |
| `@PropertySource` 로딩 | **불가** (공식 문서 명시) | 가능 |

파일명으로는 못 하는 것이 둘 있다.

```yaml
---
spring:
  config:
    activate:
      on-cloud-platform: "kubernetes"    # ← 쿠버네티스에서 돌 때만
```

```yaml
---
spring:
  config:
    activate:
      on-profile: "prod | stag"          # ← OR 조건
```

`application-prod.yml` / `application-stag.yml`로 쪼개면 **같은 내용을 두 번 써야** 한다.

### 섞어 쓰면 파일 분리가 이긴다

공식 문서가 정한 config data 적용 순서다 (**아래가 이김**).

```
1. jar 내부  application.yml              ← --- 로 나눈 on-profile 문서도 여기 소속
2. jar 내부  application-{profile}.yml
3. jar 외부  application.yml
4. jar 외부  application-{profile}.yml
```

`application.yml` 안의 `on-profile: prod` 문서와 `application-prod.yml`에 **같은 키가 있으면 `application-prod.yml`이 이긴다.** 1번과 2번이기 때문이다.

> 이건 공식 문서의 순서 목록에서 따라나오는 결론이다. 이 조합을 콕 집어 설명한 문장은 찾지 못했다.

프로파일을 여러 개 켜면 **마지막 것이 이긴다(last-wins).**

### spring.profiles.active는 아무 데나 못 쓴다

> `spring.profiles.active`와 `spring.profiles.default`는 **non-profile-specific 문서에서만** 쓸 수 있다.

```yaml
---
spring:
  config:
    activate:
      on-profile: "prod"
  profiles:
    active: "metrics"     # ✗ 기동 실패. "prod면 metrics도 켜라"는 안 된다
```

프로파일이 프로파일을 켜는 건 막혀 있다. 그런 용도로는 `spring.profiles.group`을 **맨 위 문서에** 쓴다. `spring.profiles.include`도 같은 제약을 받으며, `active`를 덮는 게 아니라 **위에 더한다.**

활성 프로파일이 하나도 없으면 `application-default`가 쓰인다.

### 어느 쪽을 쓰나

| 상황 | 선택 |
| --- | --- |
| 환경별 차이가 한두 줄 | 한 파일 `---` |
| 조건이 프로파일이 아님 (cloud platform 등) | 한 파일 `---` **밖에 없음** |
| `prod \| stag` 같은 OR | 한 파일 `---` **밖에 없음** |
| 환경별 설정이 수십 줄 | 파일 분리 |
| prod 설정만 권한·담당이 다름 | 파일 분리 |
| 배포 때 외부에서 갈아끼움 | 파일 분리 |

**둘은 배타적이지 않다.** 공통은 `application.yml`, 환경별은 별도 파일로 두고, 그 안에서 또 갈라야 할 때만 `---`를 쓰는 조합이 흔하다.

## 환경변수 이름 변환 규칙

`SPRING_PROFILES_ACTIVE`가 `spring.profiles.active`로 먹히는 이유는 relaxed binding이다. 공식 문서가 정한 변환 규칙은 셋이다.

- 점(`.`)을 밑줄(`_`)로 바꾼다
- 대시(`-`)는 **제거한다**
- 대문자로 바꾼다

```
spring.profiles.active            →  SPRING_PROFILES_ACTIVE
spring.main.log-startup-info      →  SPRING_MAIN_LOGSTARTUPINFO      ← 대시는 지워진다
achievement.consumer.user.enabled →  ACHIEVEMENT_CONSUMER_USER_ENABLED
```

대시가 **밑줄이 아니라 삭제**라는 점이 가장 자주 틀리는 부분이다. 리스트 원소는 `my.service[0].other` → `MY_SERVICE_0_OTHER`처럼 번호 양옆에 밑줄이 붙는다.

## 우선순위

공식 문서의 PropertySource 순서 중 관련 있는 것만 뽑으면 (**아래로 갈수록 우선**),

```
 1. SpringApplication.setDefaultProperties
 2. @PropertySource
 3. Config data (application.yml, application-{profile}.yml)
 5. OS 환경변수                     ← SPRING_PROFILES_ACTIVE, ACHIEVEMENT_CONSUMER_*_ENABLED
 6. Java System properties (-D)
10. SPRING_APPLICATION_JSON
11. 커맨드라인 인자                 ← --spring.profiles.active=dev
```

그래서 **커맨드라인 인자가 환경변수를 이기고, 환경변수가 yml을 이긴다.** 문서는 "command line properties always take precedence over file-based property sources"라고 못 박고 있다.

이 순서가 **환경별 배포의 핵심**이다. yml에는 안전한 기본값을 박아두고, 환경별 차이는 환경변수로 덮는다.

## 실제 배포 모양

`sooplive-achievement`의 `achievement-consumer`가 쓰는 조합이다.

```
설정 우선순위 (아래가 이김)
┌──────────────────────────────────────────────────────┐
│ application.yml            공통 기본값                │
│ application-{prof}.yml     환경별 브로커 / DB / 토픽  │
│ .env.{prof} → 환경변수     배포 시 실제 값으로 덮기   │
└──────────────────────────────────────────────────────┘
```

```yaml
# docker-compose.yml — 공통 환경변수를 YAML Anchor로 묶는다
x-consumer-env: &consumer-env
  TZ: "Asia/Seoul"
  SPRING_PROFILES_ACTIVE: ${SPRING_PROFILES_ACTIVE:-stag}
  SPRING_DATASOURCE_URL: ${SPRING_DATASOURCE_URL:-}
  JAVA_TOOL_OPTIONS: "-Xms1g -Xmx2g -XX:+UseG1GC"

services:
  achievement-consumer-user:
    environment:
      <<: *consumer-env
      ACHIEVEMENT_CONSUMER_USER_ENABLED: "true"
      ACHIEVEMENT_CONSUMER_EVENT_ENABLED: "false"

  achievement-consumer-event-catch:
    environment:
      <<: *consumer-env
      ACHIEVEMENT_CONSUMER_USER_ENABLED: "false"
      ACHIEVEMENT_CONSUMER_EVENT_ENABLED: "true"
```

```bash
docker compose --env-file .env.stag up -d
docker compose --env-file .env.prod up -d
```

정리하면 역할이 이렇게 나뉜다.

| 층 | 무엇을 정하나 |
| --- | --- |
| `SPRING_PROFILES_ACTIVE` | 어떤 `application-{profile}.yml`을 읽을지 |
| `.env.{환경}` | 브로커 주소·DB·그룹 ID 같은 **값** |
| `ACHIEVEMENT_CONSUMER_*_ENABLED` | 이 컨테이너가 **어떤 컨슈머를 켤지** |

맨 아래 줄은 `@ConditionalOnProperty`가 받는다. 같은 이미지를 여러 번 띄우되 **컨테이너마다 다른 컴포넌트만 켜지는** 구조가 여기서 나온다.

compose 쪽은 `profiles:` 키를 쓰지 않는다. 서비스를 전부 띄워놓고 환경변수로 가르기 때문에, **켜고 끄는 일을 `@ConditionalOnProperty`가 전부 담당한다.**

## 관련 문서

- [@Configuration 조건 어노테이션](configuration-conditional-annotations.md)
- [Datasource와 DSLContext](datasource_dslcontext.md)
- [@Bean](bean.md)
