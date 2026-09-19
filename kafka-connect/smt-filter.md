# SMT Filter

> 작성일: 2025-11-13

- 참조 : [Applying transformations selectively :: Debezium Documentation](https://debezium.io/documentation/reference/stable/transformations/applying-transformations-selectively.html)
- 예제

```javascript
"transforms": "onlyTitleChange,unwrap",
"transforms.onlyTitleChange.type": "io.debezium.transforms.Filter",
"transforms.onlyTitleChange.language": "cel",
"transforms.onlyTitleChange.condition": "value.op == 'u' && value.before.title_name != value.after.title_name",
"transforms.onlyTitleChange.type.behavior": "keep",
"transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
"transforms.unwrap.drop.tombstones": "true"
```

## CDC 분기처리

- 필요한 필드만 payload return

```javascript
"column.include.list": "my_db.my_info_tbl.user_name,my_db.my_info_tbl.user_age"
```
