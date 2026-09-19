# MetaData 구조

> 작성일: 2026-09-17

## 1. 계층 구조

```plain text
카탈로그 포인터 (Hive metastore / version-hint.text)
   └─ metadata.json          테이블 전체 상태
        └─ snap-*.avro       manifest list  (스냅샷당 1개)
             └─ *-m0.avro    manifest       (데이터 파일 목록 + 통계)
                  └─ *.parquet
```

한 계층이라도 없으면 아래는 못 읽습니다. 데이터 파일이 멀쩡해도 조회 실패.

## 2. 파일별 역할

| 파일 | 담는 것 | 생성 시점 |
| --- | --- | --- |
| `NNNNN-<uuid>.metadata.json` | 스키마 이력, 파티션 스펙, **스냅샷 목록**, refs, 로그 | **커밋마다 새 파일** |
| `snap-<snapid>-<seq>-<uuid>.avro` | 이 스냅샷에 속한 manifest 경로 + 파티션 범위 | 스냅샷마다 |
| `<uuid>-mNNNNN.avro` | 데이터 파일 경로, 행 수, 컬럼 min/max | 파일 그룹 변경 시 |

## 3. metadata.json 핵심 필드

```json
{
  "location": "s3://.../silver/tbl",          // 테이블 루트. 마이그레이션 시 여기부터 확인
  "current-snapshot-id": 1930815064758481733, // 지금 읽어야 할 스냅샷
  "refs": { "main": { "snapshot-id": ..., "type": "branch" } },
  "snapshots": [ { ... } ],                   // ← 배열 길이 = live 스냅샷 수
  "snapshot-log":  [ ... ],                   // 스냅샷 전환 이력
  "metadata-log":  [ ... ]                    // 과거 metadata.json 목록 (기본 100건)
}
```

snapshots 배열안에 들어있는 json

```json
{
  "snapshot-id": 1930815064758481733,
  "parent-snapshot-id": 2025521722195269963,   // 부모 체인
  "sequence-number": 467,
  "summary": { "operation": "replace",
               "manifests-replaced": "44", "manifests-created": "1" },
  "manifest-list": "s3://.../snap-1930815064758481733-1-....avro"
}
```

`operation` 읽는 법 — `append` 적재 / `delete` 삭제 / `replace`·`overwrite` 재작성 / `rewrite` 데이터 컴팩션. 위 예시는 manifest 44개를 1개로 합친 **manifest 컴팩션**이라 필요한 avro가 2개뿐인 게 정상입니다.

**snap-*.avro 헤더** (Avro 메타데이터 블록)

```plain text
snapshot-id        = 1930815064758481733   ← metadata의 current와 일치해야 함
parent-snapshot-id = 2025521722195269963
sequence-number    = 467
format-version     = 2
```
