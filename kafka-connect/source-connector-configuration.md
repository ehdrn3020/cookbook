# Source Connector 설정

> 작성일: 2025-11-19

- 예제 JSON

```javascript
"database.server.id": "1001",
"topic.prefix": "{{kafka_env}}_cdc_mariadb",
"database.include.list": "box_db",
"table.include.list": "box_db.info_tbl",
"snapshot.mode": "schema_only",

"schema.history.internal.kafka.bootstrap.servers":"{{ bootstrap_servers }}",
"schema.history.internal.kafka.topic": "{{kafka_env}}_cdc_schemahistory.box_db.info_tbl",
```

- **`dev_cdc_mariadb.box_db.info_tbl`** : 실데이터 CDC (INSERT/UPDATE/DELETE 이벤트)
- `dev_cdc_schemahistory.box_db.info_tbl` : Debezium 내부용 스키마 히스토리 저장소 / 커넥터 재시작시 과거 DDL 재적용
- `dev_cdc_mariadb` : `include.schema.changes=true`로 인해 만들어진 스키마 변경 외부용 이벤트용 토픽
