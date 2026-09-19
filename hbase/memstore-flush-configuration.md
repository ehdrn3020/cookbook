# MemStore Flush 설정

> 작성일: 2026-06-25

- `hbase.hregion.memstore.flush.size`
	- 한 Region의 MemStore가 약 `128MB`에 도달하면 flush를 요청한다.
	- MemStore 데이터를 HFile로 만들어 HDFS에 저장한다.
- `hbase.hregion.memstore.block.multiplier`
	- flush 기준의 몇 배까지 MemStore 증가를 허용할지 정한다.
	- 값이 `8`이면 쓰기 차단 기준은 약 `1GB`이다. ( 128MB × 8 = 1,024MB )
	- Region MemStore 총량이 flush.size의 8배가 되면 쓰기를 차단한다.
- 동작 흐름

```plain text
Client Put
   ↓
WAL 기록
   ↓
Active MemStore에 저장
   ↓
128MB 도달
   ↓
Flush 요청
   ↓
기존 MemStore를 Snapshot으로 전환
   ↓
HFile로 저장하는 동안
새 Active MemStore가 계속 쓰기를 받음
```

```plain text
[Snapshot 128MB] ── HFile Flush 중
        +
[Active MemStore 896MB] ── 새 Put 수신 중
        =
[Region MemStore 총량 약 1GB]

→ Put/Delete/Increment/Append 일시 대기
```

## 왜 쓰기를 차단하는가

- Flush 속도보다 쓰기 유입 속도가 빠르면 MemStore가 계속 증가할 수 있다.
- `block.multiplier`는 MemStore가 무한히 증가하지 않도록 하는 안전장치다.

```plain text
쓰기 유입 속도: 200MB/s
Flush 처리 속도: 50MB/s
순증가: 150MB/s

문제 : MemStore 과다 증가
→ JVM Heap 부족
→ Full GC 증가
→ OOM 위험
→ RegionServer 장애
```
