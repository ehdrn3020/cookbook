# Iceberg 테이블 쓰기 방식과 DataFrameWriterV2

> 작성일: 2026-11-02

- 참조 : [Spark Writes](https://iceberg.apache.org/docs/latest/spark-writes/) — Apache Iceberg 1.12.0
- 참조 : [DataFrameWriterV2 (Scala API)](https://spark.apache.org/docs/3.5.6/api/scala/org/apache/spark/sql/DataFrameWriterV2.html) — Apache Spark 3.5.6
- 참조 : [DataFrameWriterV2 (Scala API)](https://spark.apache.org/docs/4.0.0/api/scala/org/apache/spark/sql/DataFrameWriterV2.html) — Apache Spark 4.0.0

- 배치를 다시 돌려도 결과가 같아야 한다. Iceberg에 재적재하는 방법은 다섯 가지인데, 갈리는 지점은 멱등성이 아니라 **"무엇을 지우는가를 누가 정하느냐"** 다.
- `DataFrameWriterV2`(Spark 3.0)는 V1의 `mode("overwrite")` 한 단어에 뭉쳐 있던 의미를 **메서드 이름으로 쪼갠** API다.
- `INSERT OVERWRITE`는 기본값이 static이라 **`WHERE` 절과 무관하게 테이블 전체를 지운다.** 배치 코드에서 가장 조심할 지점이다.

## 목차

- [다섯 가지 방식](#다섯-가지-방식)
- [DataFrameWriterV2](#dataframewriterv2)
  - [메서드는 세 타입에 나뉘어 있다](#메서드는-세-타입에-나뉘어-있다)
  - [테이블을 만들어주지 않는다](#테이블을-만들어주지-않는다)
  - [V1 DataFrameWriter와 무엇이 다른가](#v1-dataframewriter와-무엇이-다른가)
- [overwritePartitions — 파티션 단위 재적재](#overwritepartitions--파티션-단위-재적재)
  - [빈 DataFrame은 아무것도 지우지 않는다](#빈-dataframe은-아무것도-지우지-않는다)
- [overwrite — 범위를 직접 지정](#overwrite--범위를-직접-지정)
- [INSERT OVERWRITE — 기본이 static이다](#insert-overwrite--기본이-static이다)
- [DELETE + INSERT INTO — 멱등이지만 원자적이지 않다](#delete--insert-into--멱등이지만-원자적이지-않다)
- [MERGE INTO — 행 단위 갱신](#merge-into--행-단위-갱신)
  - ["비쌈"은 상대적이다](#비쌈은-상대적이다)
  - [소스에 중복 키가 있으면 에러다](#소스에-중복-키가-있으면-에러다)
  - [WHEN NOT MATCHED BY SOURCE는 Spark 3.5부터](#when-not-matched-by-source는-spark-35부터)
- [무엇을 쓸 것인가](#무엇을-쓸-것인가)
- [버전별 차이](#버전별-차이)
- [관련 문서](#관련-문서)

## 다섯 가지 방식

| 방식 | 지우는 범위 | 지우는 범위를 정하는 주체 | 원자적 |
| --- | --- | --- | --- |
| `writeTo().overwritePartitions()` | DataFrame의 행이 속한 파티션 | **DataFrame** | ✅ |
| `writeTo().overwrite(expr)` | 조건식에 맞는 행 | **조건식** | ✅ |
| `ss.sql("INSERT OVERWRITE ...")` | 기본은 테이블 전체 | **세션 설정** | ✅ |
| `ss.sql("DELETE")` + `INSERT INTO` | 조건에 맞는 행 | 조건식 | ❌ 2스냅샷 |
| `ss.sql("MERGE INTO ...")` | 키가 매칭된 행 | **ON 절** | ✅ |

다섯 가지 모두 최종 상태로 보면 멱등이다. **멱등성은 선택 기준이 되지 못한다.** 실제로 봐야 할 것은 위 표의 3번째 칸과 4번째 칸이다.

## DataFrameWriterV2

Spark 3.0에서 추가된 DataSource V2 기반 쓰기 API다. 진입점은 `Dataset.writeTo(table: String)`.

```scala
@Experimental()
final class DataFrameWriterV2[T] extends CreateTableWriter[T]
```

`@Experimental`이 3.0부터 4.0까지 계속 붙어 있다. 정식 API지만 마이너 버전 간 호환을 보장하지 않겠다는 표시다.

### 메서드는 세 타입에 나뉘어 있다

| 출처 | 메서드 | 반환 |
| --- | --- | --- |
| `DataFrameWriterV2` | `append()` / `overwrite(condition: Column)` / `overwritePartitions()` | `Unit` |
| `CreateTableWriter` | `create()` / `replace()` / `createOrReplace()` | `Unit` |
| `CreateTableWriter` | `partitionedBy(column, columns*)` / `using(provider)` / `tableProperty(property, value)` | `CreateTableWriter[T]` |
| `WriteConfigMethods` | `option(key, value)` / `options(map)` | `DataFrameWriterV2[T]` |

반환이 `Unit`인 6개가 **종료 메서드**다. 여기서 실제 쓰기가 일어나고 체이닝이 끝난다. 나머지는 체이닝용이다.

### 테이블을 만들어주지 않는다

| 메서드 | 테이블 상태가 안 맞을 때 |
| --- | --- |
| `append()` / `overwrite()` / `overwritePartitions()` | `NoSuchTableException` |
| `create()` | `TableAlreadyExistsException` |
| `replace()` | `CannotReplaceMissingTableException` |

**쓰기 3종은 대상 테이블이 없으면 만들지 않고 예외를 던진다.** V1의 `df.write.mode("overwrite").saveAsTable(...)`이 없으면 만들어주던 것과 다르다.

배치 잡은 DDL을 먼저 적용해 두어야 하고, 이건 운영 관례가 아니라 API 제약이다.

### V1 DataFrameWriter와 무엇이 다른가

V1의 문제는 `mode("overwrite")` 한 단어가 **커넥터와 설정에 따라 다른 뜻**이었다는 점이다. 전체 교체인지 파티션 교체인지가 코드에 드러나지 않았다.

| V1 | V2 |
| --- | --- |
| `mode("append").saveAsTable(t)` | `writeTo(t).append()` |
| `mode("overwrite")` + `partitionOverwriteMode` 설정 | `writeTo(t).overwritePartitions()` |
| 없음 (설정으로만 가능) | `writeTo(t).overwrite(expr)` |
| `mode("overwrite").saveAsTable(t)` — 없으면 생성 | `create()` / `replace()` / `createOrReplace()` 로 분리 |

Iceberg 문서도 같은 얘기를 한다.

> Overwrite behavior is explicit, either dynamic or by a user-supplied filter.

`INSERT OVERWRITE`가 설정 하나로 전체 삭제가 되는 위험이 V2에는 구조적으로 없다.

## overwritePartitions — 파티션 단위 재적재

> `df.writeTo(t).overwritePartitions()` is equivalent to dynamic `INSERT OVERWRITE`.
> When Spark's overwrite mode is dynamic, partitions that have rows produced by the `SELECT` query will be replaced.

```scala
val wideDf = ss.sql(SqlLoader.load(WideSqlPath, params))   // base_dt = 2026-11-01 만 조회
wideDf.writeTo(targetTable).overwritePartitions()
```

`wideDf`에 `base_dt=2026-11-01` 행만 있으면 그 파티션 하나만 통째로 교체된다. 같은 날짜로 몇 번을 돌려도 결과가 같다.

**일배치처럼 "하루치를 통째로 다시 계산"하는 패턴에는 이게 맞는 선택이다.**

### 빈 DataFrame은 아무것도 지우지 않는다

행이 0건이면 **교체 대상 파티션도 0개**다. 기존 파티션은 그대로 남는다. 흔한 오해다.

```scala
// 이 가드의 목적은 "데이터 보호"가 아니다. overwritePartitions 는 빈 DF 로
// 파티션을 비우지 못한다. 뒤의 집계 단계를 건너뛰고 로그를 남기는 용도다.
if (wideCount == 0L) {
  println(s"No source rows for base_dt=$baseDt. Skip.")
  return
}
```

뒤집으면 이게 `overwritePartitions()`의 한계다. **이 방식으로는 파티션을 비울 수 없다.** 어제 10건이 적재됐는데 오늘 재계산 결과가 0건이 정답이라면, 그 10건은 그대로 남는다.

## overwrite — 범위를 직접 지정

```scala
import org.apache.spark.sql.functions.lit

wideDf.writeTo(targetTable).overwrite($"base_dt" === lit(baseDt))
```

**DataFrame에 뭐가 들었든 상관없이 조건에 맞는 행을 먼저 지우고 쓴다.** 그래서 위에서 말한 "0건이 정답인 경우"를 표현할 수 있다.

대가는 조건식의 책임이 전부 작성자에게 온다는 것이다. 파티션 컬럼 조건을 빼먹으면 테이블이 통째로 날아간다.

## INSERT OVERWRITE — 기본이 static이다

> Spark's default overwrite mode is **static**, but **dynamic overwrite mode is recommended when writing to Iceberg tables.**
> When Spark's overwrite mode is static, the `PARTITION` clause is converted to a filter that is used to delete from the table. **If the `PARTITION` clause is omitted, all partitions will be replaced.**

설정 없이 아래를 실행하면,

```sql
INSERT OVERWRITE catalog.gold.daily_wide
SELECT ... WHERE base_dt = DATE '2026-11-01'
```

**테이블 전체가 지워지고 하루치만 남는다.** `WHERE` 절은 지우는 범위와 아무 관계가 없다. `SELECT`의 조회 범위일 뿐이다.

dynamic으로 바꾸려면,

```properties
spark.sql.sources.partitionOverwriteMode=dynamic
```

문서는 더 나아가 `MERGE INTO` 쪽을 권한다.

> `MERGE INTO` can rewrite only affected data files and has more easily understood behavior, so it is recommended instead of `INSERT OVERWRITE`.

**세션 설정 하나에 테이블 전체가 걸리는 구조라, 배치 코드에서는 쓰지 않는 쪽이 안전하다.** 같은 일을 `overwritePartitions()`가 설정 없이 한다.

## DELETE + INSERT INTO — 멱등이지만 원자적이지 않다

최종 상태만 보면 멱등이 맞다. 문제는 **스냅샷이 두 개**라는 것이다.

```
스냅샷 N     : 기존 데이터
스냅샷 N+1   : DELETE 완료 — 읽는 쪽은 여기서 "데이터 없음"을 본다
스냅샷 N+2   : INSERT 완료
```

N+1을 읽은 소비자는 빈 결과를 받는다. 앞의 네 방식은 한 스냅샷으로 원자적으로 끝난다.

**지울 범위와 넣을 범위가 서로 다를 때**(예: 오래된 파티션 정리 + 신규 적재)는 어차피 두 연산이므로 이때만 쓴다.

`DELETE` 자체는 생각보다 싸다.

> If the delete filter matches entire partitions of the table, Iceberg will perform a **metadata-only delete**.
> If the filter matches individual rows of a table, then Iceberg will rewrite only the affected data files.

파티션 경계에 딱 맞는 `DELETE`는 데이터 파일을 건드리지 않는다. 비용이 아니라 **원자성이 문제다.**

## MERGE INTO — 행 단위 갱신

파티션을 통째로 갈아끼울 수 없을 때 쓴다. 대표적으로 **기존 행의 일부 컬럼만 갱신하고 나머지는 보존해야 하는 경우**다.

```sql
MERGE INTO catalog.achievement.action_5min_snapshot AS target
USING staging_delta AS source
ON  target.window_time = source.window_time
AND target.user_id     = source.user_id
WHEN MATCHED THEN
  UPDATE SET
    target.chat_cnt   = source.chat_cnt,
    target.updated_at = current_timestamp()
    -- applied_* 컬럼은 외부 반영 이력이므로 UPDATE SET 에서 일부러 뺀다
WHEN NOT MATCHED THEN
  INSERT (window_time, user_id, chat_cnt, applied_chat_cnt, created_at)
  VALUES (source.window_time, source.user_id, source.chat_cnt, 0, current_timestamp())
```

파티션 교체로는 `applied_*` 를 보존할 수 없다. 덮어쓰면 사라진다.

### "비쌈"은 상대적이다

파티션 교체 대비로는 비싸다. 조인이 필요하고, 영향받은 데이터 파일을 다시 쓴다.

하지만 공식 문서는 `INSERT OVERWRITE`보다 `MERGE INTO`를 권장한다. 바꿔야 할 행이 적다면 **파티션 전체를 다시 쓰는 것보다 싸다.** 어차피 파티션을 전부 다시 쓸 거라면 MERGE가 더 비싸다.

### 소스에 중복 키가 있으면 에러다

> **Only one record in the source data can update any given row of the target table**, or else an error will be thrown.

`ON` 절의 키 조합이 소스에 두 번 나오면 런타임에 실패한다. **스테이징 쿼리에서 중복 제거가 보장되는지 확인해야 한다.** 집계 쿼리라 자연히 유니크한 경우가 많지만, 코드에 그 보장이 드러나 있지 않으면 나중에 소스가 바뀌었을 때 터진다.

### WHEN NOT MATCHED BY SOURCE는 Spark 3.5부터

> Spark 3.5 added support for `WHEN NOT MATCHED BY SOURCE ... THEN ...` to update or delete rows that are not present in the source data.

"소스에 없는 타겟 행 정리"가 필요한데 Spark 3.2~3.4라면 이 구문을 쓸 수 없다. 별도 `DELETE`로 처리하거나 Spark를 올려야 한다.

## 무엇을 쓸 것인가

| 상황 | 선택 |
| --- | --- |
| 파티션 단위로 전부 다시 계산한다 | `overwritePartitions()` |
| 위와 같은데 "결과 0건"도 반영해야 한다 | `overwrite(expr)` |
| 기존 행의 일부 컬럼만 갱신하고 나머지는 보존한다 | `MERGE INTO` |
| 삭제 범위와 적재 범위가 다르다 | `DELETE` + `INSERT INTO` |
| — | `INSERT OVERWRITE` 는 쓰지 않는다 |

## 버전별 차이

| 항목 | 버전 |
| --- | --- |
| `DataFrameWriterV2` 전체 | Spark 3.0.0 |
| `MERGE INTO` 의 `WHEN NOT MATCHED BY SOURCE` | Spark 3.5 |
| `clusterBy(colName, colNames*)` | Spark 4.0 |

Spark 4.0에서 타입 선언이 바뀐다.

```scala
// 3.x
final class DataFrameWriterV2[T] extends CreateTableWriter[T]

// 4.0 — Spark Connect 와 Classic 이 같은 인터페이스를 공유하도록 분리
abstract class DataFrameWriterV2[T] extends CreateTableWriter[T]
```

전 메서드가 abstract로 바뀌었지만 **호출하는 쪽 코드는 그대로다.**

## 관련 문서

- [Spark 안의 Hive — 실행과 메타데이터 클라이언트](hive-in-spark-execution-and-metastore-client.md)
- [Spark Cache와 Persist](spark-cache-and-persist.md)
- [Spark Iceberg 예제](../iceberg/spark-iceberg-example.md)
- [Hidden Partitioning](../iceberg/hidden-partitioning.md)
- [Maintenance](../iceberg/maintenance.md)
