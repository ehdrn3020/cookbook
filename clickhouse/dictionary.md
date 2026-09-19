# DICTIONARY

> 작성일: 2026-02-10

- ClickHouse가 "OLAP 중에도 아주 빠른 Key-Value Lookup"을 하기 위해 만든 특수 기능
- **Dictionary =** JOIN 대신 쓰는, 메모리/로컬 기반 초고속 Lookup 테이블

```python
CREATE DICTIONARY user_dict
(
  user_id UInt64,
  user_grade String,
  is_vip UInt8
)
PRIMARY KEY user_id
SOURCE(...)
LAYOUT(...)
LIFETIME(...)
```
