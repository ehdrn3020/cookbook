# LRU BlockCache

> 작성일: 2026-06-23

HBase가 HFile에서 자주 읽는 데이터 블록을 메모리에 저장해두는 LRU ( Least Recently Used ) 캐시

- [RegionServer Architecture :: Apache HBase](https://hbase.apache.org/book.html#regionserver.arch)

HBase의 `LruBlockCache`는 JVM Heap을 사용하는 BlockCache 구현체

- [LruBlockCache :: Apache HBase](https://hbase.apache.org/1.4/devapidocs/org/apache/hadoop/hbase/io/hfile/LruBlockCache.html)

## 조회 흐름

- 처음 조회: HDFS 디스크에서 읽음, 다음 조회: 메모리 BlockCache에서 읽음
- 캐시가 가득 차면 가장 오래 사용되지 않은 블록(LRU)부터 제거하여 디스크 I/O를 줄여 조회 속도를 높임

```javascript
┌─────────────────┐
│ HBase Client    │
│ GET user123     │
└────────┬────────┘
         ▼
┌─────────────────────┐
│ RegionServer        │
│ 해당 Region 담당    │
└────────┬────────────┘
         ▼
┌─────────────────────────┐
│ LRU BlockCache 확인     │
│ 데이터 블록이 있는가?   │
└───────┬───────────┬─────┘
        │ Yes       │ No
        ▼           ▼
┌──────────────┐   ┌──────────────┐   ┌─────────────────┐    ┌──────────────────┐
│ Cache Hit    │   │ Cache Miss   │──▶│ DataNode:9866  │──▶│ BlockCache 저장  │
│ 메모리 읽기   │   │ HDFS 조회    │    │ HFile block 읽기│    │ 다음 조회에 사용  │
└──────┬───────┘   └──────────────┘   └─────────────────┘    └─────────┬────────┘
       └────────────────────────┬          ┬───────────────────────────┘
                                ▼          ▼
                              ┌────────────────┐
                              │ 결과 반환       │
                              └────────────────┘
```

## Scan이 문제가 될 수 있는 이유

- cache thrashing
	- 큰 테이블을 처음부터 끝까지 Scan하면 수많은 HFile 블록을 한 번씩 읽어 자주 조회하던 BlockCache가 밀려날 수 있음
- BlockCache 저장 비활성화
	- 그래서 일회성 Scan 작업에서는 필요에 따라 BlockCache 저장 비활성화 `scan.setCacheBlocks(false);`

## MemStore 와의 차이

```javascript
RegionServer Memory
├── MemStore
│   └── 아직 HFile로 flush되지 않은 쓰기 데이터
│
└── BlockCache
    └── HFile에서 읽어온 조회용 데이터 블록

MemStore
PUT
 → WAL 기록
 → MemStore 저장
 → 일정 크기가 되면 HFile로 Flush

BlockCache
GET / SCAN
 → HFile block 읽기
 → BlockCache 저장
 → 다음 조회 때 재사용

즉:
MemStore   = 쓰기 성능을 위한 메모리
BlockCache = 읽기 성능을 위한 메모리
```
