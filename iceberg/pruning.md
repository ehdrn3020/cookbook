# Pruning

> 작성일: 2026-04-07

- 쿼리할 때 불필요한 파일을 덜 읽게 만드는 기능
- Iceberg는 파일마다 이런 메타를 가지고 있음

```javascript
file1: ts min=2026-01-10, max=2026-01-11
file2: ts min=2026-01-13, max=2026-01-14

// SQL, file1은 아예 안읽음
WHERE ts = '2026-01-13'
```
