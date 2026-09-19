# ReplacingMergeTree

> 작성일: 2026-02-22

- 최신 1건만 남김
- merge 전에는 중복이 보일 수도 있어서 당장 중복 제거된 결과는 FINAL 사용
- 보통은 version 컬럼 같이 둠

```javascript
CREATE TABLE user_profile
(
  user_id UInt64,
  name String,
  age UInt8,
  ver UInt64   -- 버전(예: 업데이트 시각이나 증가값)
)
ENGINE = ReplacingMergeTree(ver)
ORDER BY user_id;

INSERT INTO user_profile VALUES
(1, 'Alice', 20, 100),
(2, 'Bob',   30, 100),
(1, 'Alice', 21, 200),
(3, 'Chris', 40, 100);

>>> ORDER BY 가 같으면 높은 ver로
| user_id | name  | age | ver |
| ------: | ----- | --: | --: |
|       1 | Alice |  21 | 200 |
|       3 | Chris |  40 | 100 |
```
