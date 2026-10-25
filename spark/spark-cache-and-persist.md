# Spark Cache와 Persist

> 작성일: 2026-10-25

- 참조 : [RDD Persistence](https://spark.apache.org/docs/latest/rdd-programming-guide.html#rdd-persistence) — StorageLevel 전체 목록과 선택 기준
- 참조 : [Caching Data](https://spark.apache.org/docs/latest/sql-performance-tuning.html#caching-data) — Spark SQL 캐시와 설정
- 참조 : [CACHE TABLE](https://spark.apache.org/docs/latest/sql-ref-syntax-aux-cache-cache-table.html) — SQL 구문
- 참조 : [Dataset (Scala API)](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/sql/Dataset.html)

- **Spark는 계산 결과를 기억하지 않는다.** 액션을 부를 때마다 처음부터 다시 계산한다.
- `cache()` / `persist()`는 그 결과를 메모리나 디스크에 붙잡아 두는 장치다.

## 1. 왜 사용하는가

### Spark는 "만드는 방법"만 들고 있다

- DataFrame은 데이터를 들고 있지 않다. **어떤 소스에서 어떤 변환을 거쳐 만들어지는지(lineage)**만 기억한다.
- 그래서 액션(`count`, `show`, `write`, `collect`…)이 불릴 때마다 **그 계보를 처음부터 다시 실행**한다.

```scala
val df = spark.read.parquet("/data/events")
  .filter($"status" === "OK")
  .withColumn("day", to_date($"ts"))

df.count()                          // 파일 읽기 + 필터 + 컬럼 추가 전부 실행
df.groupBy("day").count().show()    // ❌ 또 처음부터
df.write.parquet("/out/ok")         // ❌ 또
```

- **같은 파일을 세 번 읽는다.** 코드상으로는 `df`를 재사용한 것처럼 보이지만 실제로는 아니다.
- 이 재계산은 **에러도 경고도 없다.** 잡이 느린 이유를 찾다가 Spark UI에서 같은 스테이지가 반복되는 걸 보고 발견하게 된다.

### 캐시하면

```scala
val df = spark.read.parquet("/data/events")
  .filter($"status" === "OK")
  .withColumn("day", to_date($"ts"))
  .cache()                          // ← 추가

df.count()                          // 여기서 계산하고 결과를 저장
df.groupBy("day").count().show()    // ✅ 저장된 것에서 시작
df.write.parquet("/out/ok")         // ✅
```

- 파일 읽기와 필터가 **한 번만** 일어난다.

### 효과가 없는 경우

- **한 번만 쓰는 DataFrame.** 저장하는 비용만 더 든다. 캐시의 전제는 "두 번 이상 쓴다"이다.
- **원본을 그대로 읽기만 하는 경우.** Parquet 스캔이 이미 빠르면 캐시 이득이 작다.
- **셔플 직후.** 셔플은 이미 중간 결과를 디스크에 쓰므로 Spark가 일부 재사용한다.

## 2. 어떤 기능이 있는가

### cache()와 persist()

```scala
df.cache()                                      // 기본 레벨
df.persist()                                    // 위와 동일
df.persist(StorageLevel.MEMORY_AND_DISK_SER)    // 레벨 지정
df.unpersist()                                  // 해제
```

- **`cache()`는 `persist()`의 기본 레벨 버전**이다. 기본값이 둘로 갈린다.

| 대상 | `cache()`의 기본 레벨 |
| --- | --- |
| **RDD** | `MEMORY_ONLY` — 메모리에 안 들어가면 **버리고 재계산** |
| **DataFrame / Dataset** | `MEMORY_AND_DISK` — 넘치면 디스크로 흘림 |

- 헷갈리기 쉬운 지점이다. RDD 기준 글을 보고 "메모리 부족하면 재계산된다"고 알고 있으면 DataFrame에서는 틀린다.

### StorageLevel 종류

| StorageLevel | 저장 형태 | 메모리에 안 들어가면 |
| --- | --- | --- |
| `MEMORY_ONLY` | 역직렬화된 자바 객체 | **재계산** |
| `MEMORY_AND_DISK` | 역직렬화된 자바 객체 | 디스크로 흘림 |
| `MEMORY_ONLY_SER` | 직렬화 (파티션당 byte 배열) | **재계산** |
| `MEMORY_AND_DISK_SER` | 직렬화 | 디스크로 흘림 |
| `DISK_ONLY` | 디스크만 | — |
| `MEMORY_ONLY_2` 등 `_2` | 위와 동일 | **2개 노드에 복제** |
| `OFF_HEAP` (실험적) | 힙 밖, 직렬화 | off-heap 메모리 설정 필요 |

- `_SER`은 **공간을 아끼고 CPU를 더 쓴다.** 직렬화된 상태로 두니 메모리는 덜 먹지만 읽을 때마다 역직렬화한다.
- `_2`는 **장애 복구 속도**를 위한 것이다. 모든 레벨은 재계산으로 복구 가능하지만, 복제본이 있으면 기다리지 않고 바로 이어간다.

### 어떤 레벨을 고를까

공식문서가 제시하는 순서다.

1. **`MEMORY_ONLY`로 충분히 들어가면 그대로 둔다.** CPU 효율이 가장 좋다.
2. 안 들어가면 **`MEMORY_ONLY_SER` + 빠른 직렬화 라이브러리**를 쓴다. 공간이 크게 줄고 접근 속도는 쓸 만하다. (Java / Scala)
3. **디스크로 흘리는 건 피한다.** 계산이 아주 비싸거나 데이터를 크게 걸러내는 경우가 아니라면, 재계산이 디스크 읽기만큼 빠를 수 있다.
4. 장애 복구가 빨라야 하면 **복제 레벨(`_2`)**을 쓴다.

### unpersist()

```scala
df.unpersist()        // RDD 기준 기본은 non-blocking
df.unpersist(true)    // 해제가 끝날 때까지 대기
```

- Spark는 노드별 캐시 사용량을 감시하다가 **LRU로 오래된 파티션을 자동으로 밀어낸다.** 그래도 명시적으로 내리는 편이 좋다.
- 장기 실행 잡에서 안 내리면 **계속 쌓인다.**

### 테이블 단위 캐시

```scala
spark.catalog.cacheTable("events")        // 이름 붙은 테이블 캐시
spark.catalog.uncacheTable("events")
spark.catalog.listCachedTables()          // 캐시된 테이블 목록
```

```sql
-- SQL로도 가능
CACHE TABLE events;
CACHE LAZY TABLE events;                                    -- 처음 쓸 때 적재
CACHE TABLE hot OPTIONS ('storageLevel' 'DISK_ONLY')
  SELECT * FROM events WHERE status = 'OK';
UNCACHE TABLE events;
```

- **`CACHE TABLE`은 즉시 적재된다(eager).** `df.cache()`가 지연되는 것과 **반대**다. 미루려면 `LAZY`를 붙인다.
- `OPTIONS`를 생략하면 기본 레벨은 **`MEMORY_AND_DISK`**다.
- `listCachedTables()`에는 **이름이 붙은 것만** 나온다. `df.cache()`로만 캐시한 것은 안 보인다.

### Spark SQL 캐시는 컬럼 기반이다

- RDD 캐시와 달리 Spark SQL은 **인메모리 컬럼 포맷**으로 저장한다.
- 필요한 컬럼만 스캔하고, **컬럼별로 압축 코덱을 자동 선택**해 메모리와 GC 부담을 줄인다.

| 설정 | 기본값 | 의미 |
| --- | --- | --- |
| `spark.sql.inMemoryColumnarStorage.compressed` | `true` | 컬럼 통계를 보고 압축 코덱을 자동 선택 |
| `spark.sql.inMemoryColumnarStorage.batchSize` | `10000` | 컬럼 캐시 배치 크기. 키우면 메모리·압축에 유리하지만 **캐시 중 OOM 위험** |

## 3. 예제 코드

### 기본 — 분기되는 파이프라인

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._
import org.apache.spark.storage.StorageLevel

val spark = SparkSession.builder().appName("cache-example").getOrCreate()
import spark.implicits._

// 비싼 전처리 — 파일 스캔 + 조인 + 집계
val base = spark.read.parquet("/data/events")
  .filter($"event_dt" >= "2026-10-01")
  .join(spark.read.parquet("/data/users"), Seq("user_id"))
  .groupBy($"user_id", to_date($"ts").as("day"))
  .agg(count("*").as("cnt"), sum($"amount").as("amt"))
  .persist(StorageLevel.MEMORY_AND_DISK)

base.count()    // ① 액션 1회로 캐시를 채운다 (materialize)

// ② 아래 셋은 캐시에서 시작한다
base.groupBy($"day").agg(sum($"cnt")).write.parquet("/out/daily")
base.filter($"amt" > 10000).write.parquet("/out/whale")
base.orderBy($"cnt".desc).limit(100).write.parquet("/out/top100")

base.unpersist()    // ③ 다 쓰면 내린다
```

- **①이 중요하다.** `persist()`는 표시만 할 뿐이라, 액션이 한 번 돌아야 실제로 올라간다.
- ③을 빼먹으면 잡이 끝날 때까지 메모리를 쥐고 있는다.

### 반복 루프

```scala
// 매 반복마다 같은 데이터를 다시 읽는 경우 — 캐시 효과가 가장 큰 패턴
val train = spark.read.parquet("/data/features").cache()
train.count()

var weights = Vector.fill(10)(0.0)
for (i <- 1 to 20) {
  val grad = train.rdd
    .map(row => computeGradient(row, weights))
    .reduce(_ + _)
  weights = update(weights, grad)
}

train.unpersist()
```

- 캐시가 없으면 **20번 전부 파일을 다시 읽는다.**

### try-finally로 해제 보장

```scala
val df = spark.read.parquet("/data/events").filter($"status" === "OK").cache()
try {
  df.count()
  writeDaily(df)
  writeSummary(df)
} finally {
  df.unpersist()    // 예외가 나도 내린다
}
```

### 캐시가 실제로 올라갔는지 확인

```scala
// ① 저장 레벨 확인 — 캐시 안 걸렸으면 StorageLevel(1 replicas) 형태로 비어 있다
println(df.storageLevel)

// ② 실행 계획에 InMemoryTableScan이 보이면 캐시를 타고 있는 것
df.explain()
// == Physical Plan ==
// *(1) ColumnarToRow
// +- InMemoryTableScan [user_id#10, cnt#22L]
//       +- InMemoryRelation [...], StorageLevel(disk, memory, deserialized, 1 replicas)

// ③ 카탈로그 확인
spark.catalog.isCached("events")
```

- 가장 확실한 건 **Spark UI의 Storage 탭**이다. `Fraction Cached`가 100% 미만이면 **부분만 캐시된 것**이고, 나머지는 매번 재계산된다.

## 주의할 것

### 캐시는 지연 평가다

```scala
df.cache()      // 아무 일도 일어나지 않음 — 표시만 한다
df.count()      // 여기서 계산되고 저장된다
```

- 단 **`CACHE TABLE`(SQL)은 즉시 적재**된다. API와 SQL의 동작이 다르다.

### 남발하면 오히려 느려진다

- 캐시는 Executor의 **storage 영역**을 차지한다. 이게 커지면 **execution 영역이 밀려** 셔플 spill이 늘어난다.
- 메모리가 부족해 디스크로 흘리면 **디스크 읽기 + 역직렬화 비용**이 더해져 재계산보다 느릴 수 있다.
- → [Spark Executor OOM 원인과 해결](spark-executor-oom.md)

### 부분 캐시가 가장 헷갈린다

- 메모리가 모자라면 **일부 파티션만** 캐시된다. 캐시한 줄 알았는데 느린 전형적인 원인이다.
- `MEMORY_ONLY`면 안 들어간 파티션은 **매번 재계산**된다. Storage 탭의 `Fraction Cached`로 확인한다.

### cache와 checkpoint는 다르다

| | `cache()` / `persist()` | `checkpoint()` |
| --- | --- | --- |
| 저장 위치 | Executor 메모리·로컬 디스크 | **HDFS 등 신뢰 가능한 스토리지** |
| lineage | **유지한다** (유실 시 재계산) | **끊는다** |
| 용도 | 속도 | lineage가 길어져 생기는 문제 해결 |

- 반복이 수십 번 쌓여 계보가 길어지면 드라이버 메모리와 스케줄링에 부담이 간다. 그때는 `checkpoint()`를 섞는다.

## 관련 문서

- [Spark Executor OOM 원인과 해결](spark-executor-oom.md) — storage / execution 메모리 영역
- [SparkSql Performance Tuning](sparksql-performance-tuning.md)
- [Query Execution Plan Debugging](query-execution-plan-debugging.md) — `explain()` 읽는 법
- [Prevents Shuffle Explosions](prevents-shuffle-explosions.md)
