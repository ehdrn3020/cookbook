# Iceberg MERGE 결과를 스냅샷 비교로 검증하기

> 작성일: 2026-11-27

- 참조 : [Iceberg connector — Metadata tables, Time travel queries](https://trino.io/docs/current/connector/iceberg.html) — Trino
- 참조 : [Spark Configuration — Write options, CommitMetadata](https://iceberg.apache.org/docs/latest/spark-configuration/) — Apache Iceberg
- 참조 : [Table Spec — Snapshots](https://iceberg.apache.org/spec/) — Apache Iceberg

- 누적 테이블에 일별 증분을 MERGE로 더하는 배치는 **현재 값만 봐서는 이번에 얼마가 더해졌는지 알 수 없다.** 이전 날짜들이 이미 섞여 있기 때문이다.
- Iceberg는 커밋마다 스냅샷을 남긴다. **MERGE 커밋 직후 값 − 직전 값 = 이번 커밋이 더한 양**이고, 이걸 증분 원천(일별 집계)과 비교하면 된다.
- count 비교가 아니라 **행·컬럼 단위로** "무엇이 얼마나 바뀌었는지"까지 볼 수 있다. 쿼리는 Trino 기준이다.

## 목차

- [예제 테이블](#예제-테이블)
- [검증 대상 MERGE](#검증-대상-merge)
  - [커밋에 기준일 표시 남기기](#커밋에-기준일-표시-남기기)
- [데이터 예시](#데이터-예시)
- [1단계 — 비교할 스냅샷 찾기](#1단계--비교할-스냅샷-찾기)
- [2단계 — 증가분이 일별 집계와 같은지](#2단계--증가분이-일별-집계와-같은지)
- [3단계 — 전체 유저 불일치 건수](#3단계--전체-유저-불일치-건수)
- [4단계 — 어떤 컬럼이 바뀌었나](#4단계--어떤-컬럼이-바뀌었나)
  - [전체 유저 — 컬럼별 변경 건수](#전체-유저--컬럼별-변경-건수)
- [주의할 것](#주의할-것)
- [관련 문서](#관련-문서)

## 예제 테이블

일별 주문 집계를 유저·매장별 누적 테이블에 더하는 구조다.

```sql
-- 증분 원천: 하루치 집계 (정답지)
CREATE TABLE iceberg.demo.daily_order_agg (
    base_dt          DATE,
    user_id          VARCHAR,
    store_id         VARCHAR,
    amount           BIGINT,       -- 그날 주문 금액
    order_cnt        BIGINT,       -- 그날 주문 수
    visit_cnt        BIGINT,       -- 그날 방문 수
    first_order_dtm  TIMESTAMP(6), -- 그날 첫 주문 시각 (주문 없으면 NULL)
    last_visit_time  TIMESTAMP(6)  -- 그날 마지막 방문 시각
);

-- 누적 테이블: MERGE 대상
CREATE TABLE iceberg.demo.order_summary (
    user_id          VARCHAR,
    store_id         VARCHAR,
    total_amount     BIGINT,       -- 가산
    order_cnt        BIGINT,       -- 가산
    visit_cnt        BIGINT,       -- 가산. 나중에 추가된 컬럼이라 예전 행은 NULL
    first_order_dtm  TIMESTAMP(6), -- 처음 값 유지
    last_visit_time  TIMESTAMP(6), -- 덮어씀
    updated_at       TIMESTAMP(6)
);
```

컬럼마다 갱신 규칙이 다르다는 점이 검증의 핵심이다.

| 규칙 | 컬럼 | 검증할 때 기대값 |
| --- | --- | --- |
| 가산 | `total_amount`, `order_cnt`, `visit_cnt` | 직후 − 직전 = 그날 집계 값 |
| 처음 값 유지 | `first_order_dtm` | 직전이 NULL일 때만 바뀐다 |
| 덮어씀 | `last_visit_time` | 직후 = 그날 집계 값 |
| 항상 바뀜 | `updated_at` | 갱신된 행은 모두 바뀐다 |

## 검증 대상 MERGE

Spark SQL로 하루치를 더한다.

```sql
MERGE INTO iceberg.demo.order_summary t
USING (SELECT * FROM iceberg.demo.daily_order_agg WHERE base_dt = DATE '2026-11-25') s
ON  t.user_id = s.user_id AND t.store_id = s.store_id
WHEN MATCHED THEN UPDATE SET
    t.total_amount    = t.total_amount + s.amount,
    t.order_cnt       = t.order_cnt + s.order_cnt,
    t.visit_cnt       = coalesce(t.visit_cnt, 0) + s.visit_cnt,       -- 예전 행의 NULL 대비
    t.first_order_dtm = coalesce(t.first_order_dtm, s.first_order_dtm),
    t.last_visit_time = s.last_visit_time,
    t.updated_at      = current_timestamp()
WHEN NOT MATCHED THEN INSERT
    (user_id, store_id, total_amount, order_cnt, visit_cnt, first_order_dtm, last_visit_time, updated_at)
VALUES
    (s.user_id, s.store_id, s.amount, s.order_cnt, s.visit_cnt, s.first_order_dtm, s.last_visit_time, current_timestamp());
```

### 커밋에 기준일 표시 남기기

나중에 "25일 MERGE 커밋"을 정확히 찾으려면, 커밋할 때 스냅샷 summary에 기준일을 넣어 둔다. SQL 실행에는 `CommitMetadata.withCommitProperties`를 쓴다.

```scala
import org.apache.iceberg.spark.CommitMetadata
import org.apache.spark.sql.DataFrame
import java.util.concurrent.Callable

val props = new java.util.HashMap[String, String]()
props.put("daily-merge-run-key", baseDt)          // 예: "2026-11-25"

CommitMetadata.withCommitProperties(
  props,
  new Callable[DataFrame] { override def call(): DataFrame = spark.sql(mergeSql) },
  classOf[RuntimeException]
)
```

DataFrame 쓰기라면 write option `snapshot-property.daily-merge-run-key`로 같은 일을 할 수 있다.

## 데이터 예시

유저 `u001` 하나만 본다.

**① MERGE 직전** — 스냅샷 `1111111111111111111`

| user_id | store_id | total_amount | order_cnt | visit_cnt | first_order_dtm | last_visit_time |
| --- | --- | --- | --- | --- | --- | --- |
| u001 | s01 | 50000 | 3 | 5 | 2026-10-02 09:10 | 2026-11-24 18:20 |
| u001 | s02 | 12000 | 1 | NULL | 2026-11-01 13:00 | 2026-11-20 13:00 |

**② 25일 집계** — `daily_order_agg`, `base_dt = 2026-11-25`

| user_id | store_id | amount | order_cnt | visit_cnt | first_order_dtm | last_visit_time |
| --- | --- | --- | --- | --- | --- | --- |
| u001 | s01 | 15000 | 1 | 1 | 2026-11-25 12:30 | 2026-11-25 12:30 |
| u001 | s02 | 0 | 0 | 1 | NULL | 2026-11-25 11:00 |
| u001 | s03 | 8000 | 1 | 2 | 2026-11-25 10:05 | 2026-11-25 10:40 |

**③ MERGE 직후** — 스냅샷 `2222222222222222222`

| user_id | store_id | total_amount | order_cnt | visit_cnt | first_order_dtm | last_visit_time | 경로 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| u001 | s01 | **65000** | **4** | **6** | 2026-10-02 09:10 (유지) | **2026-11-25 12:30** | UPDATE |
| u001 | s02 | 12000 | 1 | **1** (NULL → 0+1) | 2026-11-01 13:00 | **2026-11-25 11:00** | UPDATE |
| u001 | s03 | 8000 | 1 | 2 | 2026-11-25 10:05 | 2026-11-25 10:40 | INSERT |

현재 값 ③만 보면 s01의 65000 중 얼마가 25일 몫인지 알 수 없다. **③ − ① = ②** 인지를 보는 것이 아래 쿼리들이다.

## 1단계 — 비교할 스냅샷 찾기

```sql
SELECT committed_at AT TIME ZONE 'Asia/Seoul'          AS committed_kst,
       snapshot_id,                                     -- 커밋 직후 → AFTER
       parent_id,                                       -- 커밋 직전 → BEFORE
       operation,
       element_at(summary, 'daily-merge-run-key')       AS run_key
FROM   iceberg.demo."order_summary$snapshots"
WHERE  element_at(summary, 'daily-merge-run-key') = '2026-11-25'
ORDER  BY committed_at;
```

| committed_kst | snapshot_id | parent_id | operation | run_key |
| --- | --- | --- | --- | --- |
| 2026-11-26 16:47:54.360 +0900 | 2222222222222222222 | 1111111111111111111 | overwrite | 2026-11-25 |

- `"테이블$snapshots"` 는 Iceberg 커밋 이력이다. 테이블이 바뀔 때마다 한 행씩 쌓인다. `$` 때문에 큰따옴표로 감싼다.
- `summary` 는 `map(varchar, varchar)` 다. 위에서 넣은 `daily-merge-run-key` 가 여기 들어 있다.
- `operation = overwrite` 는 MERGE가 copy-on-write로 데이터 파일을 다시 쓴 커밋이라 정상이다.
- **같은 run_key 행이 2개 이상이면 같은 날짜를 두 번 돌린 것이다.** 위 MERGE는 가산식이라 두 번 돌리면 두 번 더해진다. 배치 쪽에서 run_key로 이미 반영된 날짜인지 확인하고 건너뛰게 만들어 두는 게 좋다.

run key를 남기지 않았다면 커밋 시각으로 찾는다.

```sql
SELECT committed_at AT TIME ZONE 'Asia/Seoul' AS committed_kst, snapshot_id, parent_id, operation,
       element_at(summary, 'added-records')   AS added_records,
       element_at(summary, 'deleted-records') AS deleted_records
FROM   iceberg.demo."order_summary$snapshots"
WHERE  committed_at >= TIMESTAMP '2026-11-26 16:00:00 Asia/Seoul'
ORDER  BY committed_at;
```

## 2단계 — 증가분이 일별 집계와 같은지

1단계의 `snapshot_id` 와 `parent_id` 를 **숫자 그대로** 넣는다.

```sql
WITH s_after AS (
    SELECT * FROM iceberg.demo.order_summary FOR VERSION AS OF 2222222222222222222
    WHERE user_id = 'u001'
),
s_before AS (
    SELECT * FROM iceberg.demo.order_summary FOR VERSION AS OF 1111111111111111111
    WHERE user_id = 'u001'
),
agg AS (
    SELECT * FROM iceberg.demo.daily_order_agg
    WHERE base_dt = DATE '2026-11-25' AND user_id = 'u001'
)
SELECT g.store_id,
       CASE WHEN b.store_id IS NULL THEN 'NEW' ELSE 'UPD' END              AS kind,
       -- 왼쪽: summary가 늘어난 양 / 오른쪽: 그날 집계. 쌍마다 같아야 한다
       a.total_amount - coalesce(b.total_amount, 0)                          AS d_amount, g.amount,
       a.order_cnt    - coalesce(b.order_cnt, 0)                             AS d_order,  g.order_cnt,
       coalesce(a.visit_cnt, 0) - coalesce(b.visit_cnt, 0)                   AS d_visit,  g.visit_cnt,
       a.last_visit_time                                                     AS summary_last_visit,
       g.last_visit_time                                                     AS agg_last_visit
FROM   agg g
LEFT   JOIN s_after  a ON a.store_id = g.store_id
LEFT   JOIN s_before b ON b.store_id = g.store_id
ORDER  BY g.store_id;
```

| store_id | kind | d_amount | amount | d_order | order_cnt | d_visit | visit_cnt | summary_last_visit | agg_last_visit |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| s01 | UPD | 15000 | 15000 | 1 | 1 | 1 | 1 | 2026-11-25 12:30 | 2026-11-25 12:30 |
| s02 | UPD | 0 | 0 | 0 | 0 | 1 | 1 | 2026-11-25 11:00 | 2026-11-25 11:00 |
| s03 | NEW | 8000 | 8000 | 1 | 1 | 2 | 2 | 2026-11-25 10:40 | 2026-11-25 10:40 |

| 부분 | 역할 |
| --- | --- |
| `FOR VERSION AS OF` | Iceberg time travel. 이후 26일 배치로 테이블이 더 바뀌었어도 정확히 그 커밋 시점을 읽는다 |
| `agg` 기준 `LEFT JOIN` | 직전에 없던 쌍은 `b`가 NULL → `NEW`(INSERT 경로). summary에 반영이 안 된 쌍이 있으면 `a`가 NULL로 드러난다 |
| `coalesce(b.xxx, 0)` | 신규 쌍은 직전 값을 0으로 본다 |
| `coalesce(a.visit_cnt, 0)` | 나중에 추가된 컬럼은 예전 행에 NULL이 있다 |
| `last_visit_time` 비교 | 덮어쓰는 컬럼은 차이가 아니라 **값이 같은지**를 본다 |

## 3단계 — 전체 유저 불일치 건수

유저 조건을 빼고 건수만 센다. 이번엔 조인 키가 `user_id + store_id` 둘 다여야 한다.

```sql
WITH s_after  AS (SELECT * FROM iceberg.demo.order_summary FOR VERSION AS OF 2222222222222222222),
     s_before AS (SELECT * FROM iceberg.demo.order_summary FOR VERSION AS OF 1111111111111111111),
     agg      AS (SELECT * FROM iceberg.demo.daily_order_agg WHERE base_dt = DATE '2026-11-25')
SELECT count(*)                                    AS agg_pairs,
       count(*) FILTER (WHERE a.store_id IS NULL)  AS missing_in_summary,   -- 0 이어야 함
       count(*) FILTER (WHERE b.store_id IS NULL)  AS new_pairs,
       count(*) FILTER (WHERE
             a.total_amount - coalesce(b.total_amount, 0)        <> g.amount
          OR a.order_cnt    - coalesce(b.order_cnt, 0)           <> g.order_cnt
          OR coalesce(a.visit_cnt, 0) - coalesce(b.visit_cnt, 0) <> g.visit_cnt
          OR a.last_visit_time IS DISTINCT FROM g.last_visit_time
       )                                           AS mismatch              -- 0 이어야 함
FROM   agg g
LEFT   JOIN s_after  a ON a.user_id = g.user_id AND a.store_id = g.store_id
LEFT   JOIN s_before b ON b.user_id = g.user_id AND b.store_id = g.store_id;
```

| agg_pairs | missing_in_summary | new_pairs | mismatch |
| --- | --- | --- | --- |
| 15230 | 0 | 2104 | 0 |

`missing_in_summary = 0` 이고 `mismatch = 0` 이면 25일 집계가 누락 없이 정확히 반영된 것이다. 0이 아니면 같은 조건으로 `SELECT g.*, a.*, b.*` 를 뽑아 행을 직접 본다.

## 4단계 — 어떤 컬럼이 바뀌었나

UPD 행을 컬럼별로 펼쳐서 직전 값, 직후 값, 변경 여부를 한 줄씩 본다.

```sql
WITH s_after AS (
    SELECT * FROM iceberg.demo.order_summary FOR VERSION AS OF 2222222222222222222
    WHERE user_id = 'u001'
),
s_before AS (
    SELECT * FROM iceberg.demo.order_summary FOR VERSION AS OF 1111111111111111111
    WHERE user_id = 'u001'
),
pairs AS (
    SELECT a.store_id,
           ARRAY[
             ROW('total_amount',    CAST(b.total_amount    AS varchar), CAST(a.total_amount    AS varchar)),
             ROW('order_cnt',       CAST(b.order_cnt       AS varchar), CAST(a.order_cnt       AS varchar)),
             ROW('visit_cnt',       CAST(b.visit_cnt       AS varchar), CAST(a.visit_cnt       AS varchar)),
             ROW('first_order_dtm', CAST(b.first_order_dtm AS varchar), CAST(a.first_order_dtm AS varchar)),
             ROW('last_visit_time', CAST(b.last_visit_time AS varchar), CAST(a.last_visit_time AS varchar)),
             ROW('updated_at',      CAST(b.updated_at      AS varchar), CAST(a.updated_at      AS varchar))
           ] AS cols
    FROM   s_after a
    JOIN   s_before b ON b.store_id = a.store_id          -- INNER JOIN: 직전에도 있던 쌍(UPD)만
)
SELECT p.store_id,
       c.col_nm,
       c.before_val,
       c.after_val,
       CASE WHEN c.before_val IS NOT DISTINCT FROM c.after_val THEN '' ELSE 'CHANGED' END AS changed
FROM   pairs p
CROSS  JOIN UNNEST(p.cols) AS c(col_nm, before_val, after_val)
ORDER  BY p.store_id, changed DESC, c.col_nm;
```

| store_id | col_nm | before_val | after_val | changed |
| --- | --- | --- | --- | --- |
| s01 | last_visit_time | 2026-11-24 18:20:00.000000 | 2026-11-25 12:30:00.000000 | CHANGED |
| s01 | order_cnt | 3 | 4 | CHANGED |
| s01 | total_amount | 50000 | 65000 | CHANGED |
| s01 | updated_at | 2026-11-25 16:41:02.118000 | 2026-11-26 16:47:53.902000 | CHANGED |
| s01 | visit_cnt | 5 | 6 | CHANGED |
| s01 | first_order_dtm | 2026-10-02 09:10:00.000000 | 2026-10-02 09:10:00.000000 | |
| s02 | last_visit_time | 2026-11-20 13:00:00.000000 | 2026-11-25 11:00:00.000000 | CHANGED |
| s02 | updated_at | 2026-11-20 16:40:11.530000 | 2026-11-26 16:47:53.902000 | CHANGED |
| s02 | visit_cnt | NULL | 1 | CHANGED |
| s02 | first_order_dtm | 2026-11-01 13:00:00.000000 | 2026-11-01 13:00:00.000000 | |
| s02 | order_cnt | 1 | 1 | |
| s02 | total_amount | 12000 | 12000 | |

| 장치 | 이유 |
| --- | --- |
| `ARRAY[ROW(...)]` + `UNNEST` | 컬럼을 행으로 펼친다 (unpivot). 컬럼이 늘어도 `ROW` 한 줄만 추가하면 된다 |
| `CAST(... AS varchar)` | 숫자·시각이 섞여 있어 배열 원소 타입을 맞춘다 |
| `IS NOT DISTINCT FROM` | NULL과 NULL을 같다고 본다. 그래서 s02의 `visit_cnt` NULL → 1도 CHANGED로 잡힌다 |
| `INNER JOIN s_before` | NEW 행은 직전 값이 없어 비교할 게 없으니 뺀다 |

읽는 법:

- s02는 그날 주문이 없어서 `amount = 0` 이다. MERGE는 `total_amount + 0` 으로 다시 쓰지만 **값이 같으니 CHANGED로 잡히지 않는다.**
- `first_order_dtm` 은 두 행 모두 직전 값이 있어서 유지됐다. `coalesce(target, source)` 규칙대로다.
- 가산 규칙인 컬럼이 delta 0인데도 CHANGED로 나오면, 직전 값이 NULL이었다가 `coalesce(t.x, 0) + 0` 으로 0이 채워진 경우다. 정상이다.

### 전체 유저 — 컬럼별 변경 건수

유저 조건을 빼고 컬럼별로 몇 건이 바뀌었는지 센다.

```sql
WITH s_after  AS (SELECT * FROM iceberg.demo.order_summary FOR VERSION AS OF 2222222222222222222),
     s_before AS (SELECT * FROM iceberg.demo.order_summary FOR VERSION AS OF 1111111111111111111),
pairs AS (
    SELECT ARRAY[
             ROW('total_amount',    CAST(b.total_amount    AS varchar), CAST(a.total_amount    AS varchar)),
             ROW('order_cnt',       CAST(b.order_cnt       AS varchar), CAST(a.order_cnt       AS varchar)),
             ROW('visit_cnt',       CAST(b.visit_cnt       AS varchar), CAST(a.visit_cnt       AS varchar)),
             ROW('first_order_dtm', CAST(b.first_order_dtm AS varchar), CAST(a.first_order_dtm AS varchar)),
             ROW('last_visit_time', CAST(b.last_visit_time AS varchar), CAST(a.last_visit_time AS varchar)),
             ROW('updated_at',      CAST(b.updated_at      AS varchar), CAST(a.updated_at      AS varchar))
           ] AS cols
    FROM   s_after a
    JOIN   s_before b ON b.user_id = a.user_id AND b.store_id = a.store_id
)
SELECT c.col_nm,
       count(*) FILTER (WHERE c.before_val IS DISTINCT FROM c.after_val) AS changed_rows,
       count(*)                                                         AS compared_rows
FROM   pairs p
CROSS  JOIN UNNEST(p.cols) AS c(col_nm, before_val, after_val)
GROUP  BY c.col_nm
ORDER  BY changed_rows DESC;
```

| col_nm | changed_rows | compared_rows |
| --- | --- | --- |
| updated_at | 13126 | 4180512 |
| last_visit_time | 13126 | 4180512 |
| visit_cnt | 13126 | 4180512 |
| total_amount | 9840 | 4180512 |
| order_cnt | 9840 | 4180512 |
| first_order_dtm | 37 | 4180512 |

- `compared_rows` 는 직전·직후에 모두 있는 쌍 전체다. 25일에 손대지 않은 쌍도 들어간다.
- **`updated_at` 의 changed_rows = 이번에 UPDATE된 행 수**다. 3단계의 `agg_pairs − new_pairs`(15230 − 2104 = 13126)와 같아야 한다.
- `total_amount` 가 그보다 적은 건 방문만 하고 주문은 없던 쌍(delta 0)이 있어서다.
- `first_order_dtm` 이 37건뿐인 건 직전에 NULL이던(그 전까지 주문이 없던) 쌍만 바뀌기 때문이다.

## 주의할 것

| 항목 | 내용 |
| --- | --- |
| 스냅샷 ID 자리 | `<AFTER>` 같은 자리표시를 그대로 실행하면 `mismatched input '<'` 문법 오류가 난다. 숫자를 넣는다 |
| `FOR VERSION AS OF` 에 서브쿼리 | Trino 문서 예시는 모두 리터럴이다. 그래서 1단계에서 ID를 찾고 2단계에 붙여 넣는 두 단계로 나눴다 «서브쿼리 가능 여부는 확인 안 함» |
| 같은 날짜를 다시 돌린 커밋 | 그 커밋의 직전·직후를 비교하면 "다시 돌린 영향"만 보인다. 첫 반영을 검증하려면 **그날의 첫 커밋**을 골라야 한다 |
| 스냅샷 만료 | `expire_snapshots` 는 보존 기간이 지난 스냅샷과 관련 파일을 지운다. 검증할 커밋이 만료되면 time travel로 읽을 수 없으니, 검증은 만료 주기 안에 끝낸다 |
| 시간대 | `committed_at` 은 `timestamp with time zone` 이다. `AT TIME ZONE 'Asia/Seoul'` 로 바꿔 봐야 배치 로그 시각과 맞춰 보기 쉽다 |

## 관련 문서

- [MetaData 구조](metadata-structure.md) — 스냅샷과 manifest의 관계
- [Maintenance](maintenance.md) — Expire Snapshots
- [Branching & Tagging](branching-and-tagging.md) — 검증할 스냅샷을 만료되지 않게 tag로 고정하기
- [Iceberg 테이블 쓰기 방식과 DataFrameWriterV2](../spark/iceberg-table-write-modes-and-dataframewriterv2.md) — MERGE INTO를 쓰는 경우
