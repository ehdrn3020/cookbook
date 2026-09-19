# HappyBase Example

> 작성일: 2025-12-29

```javascript
import happybase
con = happybase.Connection('thriftserver.ip.or.domain')
user_id = '123123'
table = con.table('users_tbl')
row_key = user_id.encode()

# 모든 row 컬럼 최신값 -> get 'users_tbl', '123123'
row = table.row(row_key)

# 단일 컬럼과 버전 값 -> get 'users_tbl', '123123', { COLUMN => 'info:nickname', VERSIONS => 3 }
column = 'info:nickname'.encode()
cells = table.cells(row_key, column, versions=10, include_timestamp=True)

for col, value in row.items():
	print(col.decode(), value.decode('utf-8'))
```
