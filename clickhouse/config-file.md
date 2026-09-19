# Config File

> 작성일: 2026-03-29

- `clickhouse-server/config.d/config.xml` : 서버/클러스터 설정
- `clickhouse-server/users.d/users.xml` : 사용자/권한/프로필/쿼터 설정
- `clickhouse-keeper/keeper_config.xml` : ClickHouse Keeper 자체 설정

## Merging Configuration

- `/etc/clickhouse-server/config.xml`을 쓰고, 추가 설정은 `config.d/` 아래에 여러 파일
- ClickHouse는 이 여러 파일을 읽어서 **하나의 최종 설정으로 병합**
- `<default replace="replace">`, `<config_c remove="remove">` 사용하여 겹치는 내용 처리

## Substitution by environment variables ( and zookeeper )

- 설정값을 XML 안에 직접 쓰는 대신, **외부 값으로 치환**하는 기능
	- Docker나 Kubernetes 편리하게 활용
- 환경변수 치환: `from_env`

```plain text
<max_query_sizefrom_env="MAX_QUERY_SIZE"/>
>>>
환경변수 MAX_QUERY_SIZE=150000 이 잡혀 있으면 실제 설정은:
<max_query_size>150000</max_query_size>
```

- ClickHouse Keeper 치환: `from_zk`

```plain text
<postgresql_portfrom_zk="/zk_configs/postgresql_port"/>
```
