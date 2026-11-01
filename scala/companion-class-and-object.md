# 컴패니언 클래스와 컴패니언 오브젝트

> 작성일: 2026-11-01

- 참조 : [Singleton Objects — Companion Objects](https://docs.scala-lang.org/tour/singleton-objects.html) — Scala Documentation
- 참조 : [Case Classes](https://docs.scala-lang.org/tour/case-classes.html) — Scala Documentation
- 참조 : [String Interpolation](https://docs.scala-lang.org/scala3/book/string-interpolation.html) — Scala Documentation
- 참조 : [Main Methods](https://docs.scala-lang.org/scala3/book/methods-main-methods.html) — Scala Documentation
- 참조 : [scala.Predef — require](https://www.scala-lang.org/api/2.13.x/scala/Predef$.html) — Scala 2.13 API

- 같은 파일에 같은 이름으로 정의한 `class`와 `object`는 서로 **컴패니언(companion)** 관계가 된다.
- 하나의 개념을 **"인스턴스마다 다른 것"**과 **"모두가 공유하는 것"**으로 나누어 담는 구조다.
- Java로 치면 하나의 클래스를 인스턴스 멤버와 `static` 멤버로 나눈 것과 같다. Scala에는 `static` 키워드가 없고, 그 자리가 컴패니언 오브젝트다.

> 이 문서의 예제는 설명용으로 작성한 것이며 Scala 2.13 기준 문법이다. **직접 컴파일·실행해 확인하지는 않았다.**

## 목차

- [전체 예제](#전체-예제)
- [컴패니언이란](#컴패니언이란)
- [Java의 static과 비교](#java의-static과-비교)
- [of 팩토리 자세히 보기](#of-팩토리-자세히-보기)
  - [메서드 시그니처](#메서드-시그니처)
  - [require와 문자열 보간](#require와-문자열-보간)
  - [객체는 언제 생성되나](#객체는-언제-생성되나)
  - [of로만 생성하게 강제하기](#of로만-생성하게-강제하기)
- [언제 컴패니언 오브젝트를 쓰나](#언제-컴패니언-오브젝트를-쓰나)
- [실행할 때 주의할 점](#실행할-때-주의할-점)
  - [.scala 파일에는 진입점이 필요하다](#scala-파일에는-진입점이-필요하다)
  - [Worksheet(.sc)와 .scala의 차이](#worksheetsc와-scala의-차이)
- [관련 문서](#관련-문서)

## 전체 예제

파일 위치 예: `src/main/scala/TargetUserConfig.scala`

```scala
// ① 컴패니언 클래스: 값을 담는 그릇 (인스턴스)
case class TargetUserConfig(minBroadSec: Long, targetDays: Int)

// ② 컴패니언 오브젝트: 그릇을 만드는 공장 (싱글톤)
object TargetUserConfig {
  val DefaultMinBroadSec: Long = 3600L

  def fromArgs(args: Map[String, String]): TargetUserConfig =
    of(
      args.getOrElse("minBroadSec", DefaultMinBroadSec.toString).toLong,
      args.getOrElse("targetDays", "30").toInt
    )

  // 검증 후 생성하는 팩토리 (Java의 static of 자리)
  def of(minBroadSec: Long, targetDays: Int): TargetUserConfig = {
    require(minBroadSec >= 0, s"minBroadSec는 0 이상이어야 합니다: $minBroadSec")
    require(targetDays > 0, s"targetDays는 양수여야 합니다: $targetDays")
    TargetUserConfig(minBroadSec, targetDays)
  }
}

// ③ 실행 진입점
object Main {
  def main(args: Array[String]): Unit = {
    println(TargetUserConfig.of(7200L, 14))                       // TargetUserConfig(7200,14)
    println(TargetUserConfig.fromArgs(Map("targetDays" -> "7")))  // TargetUserConfig(3600,7)
    println(TargetUserConfig.DefaultMinBroadSec)                  // 3600
    // TargetUserConfig.of(3600L, 0)  // IllegalArgumentException 발생
  }
}
```

## 컴패니언이란

공식 문서의 정의는 두 문장이다.

> "An object with the same name as a class is called a **companion object**."
>
> "If a class or object has a companion, both must be defined in the **same file**."

`case class TargetUserConfig`와 `object TargetUserConfig`가 같은 파일에 같은 이름으로 있으므로 둘은 한 쌍이 된다. **이름이 같아도 다른 파일에 있으면 컴패니언이 아니다.**

| 구분 | ① `case class` | ② `object` |
| --- | --- | --- |
| 개수 | 생성할 때마다 여러 개 | 프로그램 전체에 딱 하나 |
| 담는 것 | 각자의 값 (`minBroadSec`, `targetDays`) | 모두가 공유하는 것 (`DefaultMinBroadSec`, `fromArgs`, `of`) |
| 호출 방식 | `conf.minBroadSec` (인스턴스로) | `TargetUserConfig.of(...)` (이름으로 바로) |
| Java로 치면 | 인스턴스 필드·메서드 | `static` 필드·메서드 |

컴패니언 관계가 되면 다음 효과가 생긴다.

- **서로의 `private` 멤버에 접근할 수 있다.** 공식 문서 표현으로 "A companion class or object can access the private members of its companion." 클래스 생성자를 `private`으로 막고 오브젝트의 팩토리로만 생성하게 강제할 수 있다
- **관련 기능이 같은 이름 아래 모인다.** `TargetUserConfig.DefaultMinBroadSec`, `TargetUserConfig.of`처럼 읽기 쉽다

공식 문서의 예제는 `case class Circle`과 `object Circle` 쌍이다. 오브젝트에 `private`으로 둔 `calculateArea`를 클래스가 `import Circle._`로 끌어다 쓴다.

## Java의 static과 비교

Scala에는 `static` 키워드가 없다. Java에서 `static`을 붙이는 기준은 **"이 멤버가 특정 객체의 필드 값을 쓰는가?"**이다. 쓰지 않는 멤버는 객체마다 따로 가질 필요가 없으므로 클래스에 하나만 두는데, Scala에서는 그 자리가 컴패니언 오브젝트다.

```java
// Java
public class TargetUserConfig {
    private final long minBroadSec;                         // 인스턴스 멤버
    private final int targetDays;

    public static final long DEFAULT_MIN_BROAD_SEC = 3600L; // static 상수
    public static TargetUserConfig of(long minBroadSec, int targetDays) { ... } // static 팩토리
}
```

```scala
// Scala
case class TargetUserConfig(minBroadSec: Long, targetDays: Int)     // 인스턴스 멤버

object TargetUserConfig {
  val DefaultMinBroadSec: Long = 3600L                               // static 상수 자리
  def of(minBroadSec: Long, targetDays: Int): TargetUserConfig = ... // static 팩토리 자리
}
```

Java의 `Map.of(k, v)`가 `Map` 인터페이스의 static 팩토리이듯, Scala의 `Map(k -> v)`, `List(1, 2, 3)`도 각 컴패니언 오브젝트의 `apply` 팩토리를 호출하는 것이다.

## of 팩토리 자세히 보기

### 메서드 시그니처

```scala
def of(minBroadSec: Long, targetDays: Int): TargetUserConfig = { ... }
//  ①   ②                                   ③                  ④
```

①은 메서드 이름, ②는 파라미터(`이름: 타입`), ③은 **반환 타입**, ④는 본문이다. Scala는 타입을 이름 뒤에 콜론으로 붙이므로 반환 타입도 괄호 뒤에 온다. Java의 `public static TargetUserConfig of(long minBroadSec, int targetDays)`와 같다.

`return` 키워드는 쓰지 않는다. 블록의 **마지막 표현식 값**이 반환된다. 반환값이 없는 메서드의 반환 타입은 `Unit`이며, Java의 `void`에 해당한다.

### require와 문자열 보간

```scala
require(minBroadSec >= 0, s"minBroadSec는 0 이상이어야 합니다: $minBroadSec")
```

`scala.Predef`의 시그니처는 둘이다.

```scala
final def require(requirement: Boolean, message: => Any): Unit
def require(requirement: Boolean): Unit
```

조건이 `false`이면 `IllegalArgumentException`을 던진다. 공식 API 문서는 `assert`와의 차이를 이렇게 적고 있다.

> `require`는 **조건을 어긴 호출자를 탓한다(blames the caller).**

즉 **인자 검증**에는 `require`, 내부 불변식 검증에는 `assert`다.

`message` 파라미터의 타입이 `=> Any`인 점이 중요하다. **by-name 파라미터라 조건이 실패했을 때만 평가된다.** 정상 통과할 때는 보간 문자열을 만드는 비용이 없다.

실패하면 이런 예외가 난다.

```
java.lang.IllegalArgumentException: requirement failed: minBroadSec는 0 이상이어야 합니다: -5
```

> 메시지 앞의 `requirement failed:`는 Scala가 붙여 주는 것이다. 다만 **API 문서에 이 접두사가 명시돼 있지는 않다** — 구현 동작이다.

문자열 앞의 `s`는 **문자열 보간(string interpolation)**이다.

| 표기 | 의미 |
| --- | --- |
| `s"...$name..."` | `name`의 `toString()` 결과로 치환 |
| `s"...${2 + 2}..."` | 중괄호 안에 **임의의 식**을 넣을 수 있다 |
| `"...$name..."` | **`s`가 없으면 치환되지 않는다.** 글자 그대로 출력 |
| `s"...$$..."` | 달러 기호 자체 |
| `f"$height%2.2f"` | `printf` 형식. 타입이 맞지 않으면 **컴파일 에러** |
| `raw"a\n$foo"` | 치환은 하되 `\n` 같은 이스케이프는 처리하지 않는다 |

### 객체는 언제 생성되나

`of`를 호출할 때마다 검증을 통과하면 새 인스턴스가 하나씩 생성된다. 실제 생성은 마지막 줄 `TargetUserConfig(minBroadSec, targetDays)`에서 일어난다.

공식 문서 표현으로 "case classes have an `apply` method by default which takes care of object construction" — 컴파일러가 컴패니언에 자동으로 넣어 준 `apply`를 부르는 것이고, `apply`가 내부에서 `new`를 실행한다. **검증에 실패하면 객체는 만들어지지 않는다.**

```scala
val a = TargetUserConfig.of(3600L, 7)
val b = TargetUserConfig.of(3600L, 7)

a == b   // true  : case class는 필드 값으로 비교
a eq b   // false : 메모리상 서로 다른 객체
```

`==`가 `true`인 근거도 공식 문서에 있다.

> "Instances of case classes are compared by **structure** and not by reference."

| 대상 | 개수 |
| --- | --- |
| `object TargetUserConfig` (공장) | 항상 1개 (처음 접근할 때 생성) |
| `of`로 만든 인스턴스 (제품) | 호출할 때마다 1개씩 |

case class 파라미터는 **public `val`**이라 재할당할 수 없다. 분산 환경에서 안전한 불변 객체가 되는 이유다.

### of로만 생성하게 강제하기

지금 구조에서는 `TargetUserConfig(3600L, 0)`처럼 직접 생성하면 검증을 건너뛸 수 있다. 생성자를 `private`으로 바꾸면 컴패니언 오브젝트(`of`)를 통해서만 만들 수 있다.

```scala
case class TargetUserConfig private (minBroadSec: Long, targetDays: Int)
```

다만 **Scala 버전에 따라 case class의 `apply`나 `copy`로 우회할 수 있는 경우가 있다.** 쓰는 버전에서 직접 컴파일해 확인해야 한다.

## 언제 컴패니언 오브젝트를 쓰나

판단 기준은 **"이 기능을 쓰려면 인스턴스부터 만들어야 하는가?"**이다. 인스턴스가 있어야 의미 있는 것(각자의 필드, 그 값을 쓰는 메서드)은 클래스에 둔다. 인스턴스 없이 쓰는 것은 컴패니언 오브젝트에 둔다.

| 용도 | 예 |
| --- | --- |
| 팩토리 (생성 방법이 여러 가지이거나 검증이 필요할 때) | `of`, `fromArgs`, `default` |
| 상수·기본값 | `DefaultMinBroadSec` |
| 그 타입을 다루는 공용 도구 | 테이블에서 읽어오는 `load`, JSON 변환기, implicit 인코더 |

객체를 만드는 메서드는 **호출 시점에 아직 객체가 없으므로** 인스턴스 메서드가 될 수 없다. 그래서 팩토리가 컴패니언 오브젝트의 가장 흔한 용도다.

반대로 다음 경우에는 직접 만들 필요가 없다.

- 값만 담는 단순한 case class — 컴파일러가 `apply`, `unapply`, `copy`를 이미 만들어 준다
- 특정 클래스와 무관한 유틸리티(날짜 변환 함수 모음 등) — 이름을 맞추지 않은 **독립 `object`**로 두면 된다

## 실행할 때 주의할 점

### .scala 파일에는 진입점이 필요하다

Scala 2의 `.scala` 파일 최상위에는 `class`, `object`, `trait` 정의만 올 수 있다. `val conf = ...`나 `conf.minBroadSec` 같은 실행문을 최상위에 두면 이런 에러가 난다.

```
expected class or object definition
```

JVM은 프로그램을 시작할 때 `static void main(String[] args)`를 찾는다. Scala에서는 `object` 안의 `def main(args: Array[String]): Unit`이 그 역할을 하므로, 실행 코드는 `object Main`의 `main` 안에 넣는다. 오브젝트 이름은 `Main`이 아니어도 된다.

버전별로 진입점 작성법이 다르다.

| 방법 | 버전 |
| --- | --- |
| `object X { def main(args: Array[String]): Unit = ... }` | **Scala 2 / 3 모두.** 크로스 빌드가 필요하면 공식 문서가 이 방식을 권한다 |
| `object X extends App` | Scala 2 방식. Scala 3에서는 기능이 제한되고 **앞으로 deprecated 예정**이라고 공식 문서가 밝히고 있다 |
| `@main def run(...) = ...` | **Scala 3 전용.** Scala 2에서 쓰면 `Cannot resolve symbol main` |

Scala 3에서는 최상위 정의 자체가 허용되어("all kinds of definitions can be written at the top level") 이 제약이 사라졌다. **Scala 2에서만 걸리는 문제다.**

### Worksheet(.sc)와 .scala의 차이

| 구분 | Worksheet (`.sc`) | Scala 파일 (`.scala`) |
| --- | --- | --- |
| 용도 | 문법 실험, 값 확인 | 실제 애플리케이션 코드 |
| 최상위 실행문 | 가능 | 불가 (Scala 2) |
| 진입점 | 불필요, 위에서부터 평가 | `def main` 필요 |
| 결과 확인 | 오른쪽 패널에 각 줄의 값·타입 표시 | `println`·로그로 출력 |
| 빌드 포함 | 포함 안 됨 | 컴파일되어 jar에 포함 |

짧게 확인하고 버릴 코드는 Worksheet에서, 빌드해서 실행하거나 다른 코드에서 쓸 코드는 `.scala` 파일에서 작성한다.

## 관련 문서

- [CaseClass](case-class.md)
- [SBT IntelliJ 환경](sbt-intellij-setup.md)
