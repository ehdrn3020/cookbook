# OP 종류

> 작성일: 2025-11-12

| op | 의미 | 언제 발생 | before/after |
| --- | --- | --- | --- |
| `r` | **스냅샷 읽기(Read)** | 스냅샷 중 테이블 **현재 상태**를 읽어 보낼 때 | 보통 `before=null`, `after`에 **행의 현 상태** |
| `c` | **생성(Create)** | INSERT 감지 시 | `before=null`, `after`에 신규 값 |
| `u` | **수정(Update)** | UPDATE 감지 시 | `before`에 이전 값, `after`에 변경된 값 |
| `d` | **삭제(Delete)** | DELETE 감지 시 | `before`에 삭제 전 값, `after=null` |
| `t` | **테이블 단위 삭제(Truncate)** | 일부 커넥터/DB에서 TRUNCATE 감지 시 | 메타 성격(행 단위 before/after 없음) |
