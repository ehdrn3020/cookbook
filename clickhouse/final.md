# FINAL

> 작성일: 2026-01-23

- ReplacingMergeTree engine에서 백그라운드 merge가 아직 안 끝났더라도, 쿼리 시점에 논리적으로 '정리된 최종 상태'를 만들어서 읽음
- 중복제거, 삭제, 예전 값이 남아있는 row 제거하고 보여줌

```javascript
SELECT * FROM aa;
1 A 1
1 B 2
SELECT * FROM aa FINAL;
1 B 2
-> 최신 version만 남김
```
