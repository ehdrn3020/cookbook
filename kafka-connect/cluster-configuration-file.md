# Cluster 설정파일

> 작성일: 2025-11-17

/confluent_home/etc/kafka/connect-distributed.properties

```javascript
# Kafka Connect 워커 프로세스가 처음에 Kafka 클러스터에 접속할 때 사용하는 브로커 주소 목록
bootstrap.servers={{ bootstrap_servers }}

# Kafka Connect "클러스터"를 구분하는 ID
group.id={{ group_id }}

# Kafka Connect가 커넥터/태스크 오프셋을 내부 offset 저장소 토픽에 얼마마다 기록할지를 설정
offset.flush.interval.ms=10000

# kafka connect 메세지의 json 형식을 수정(스키마 + 페이로드)
key.converter.schemas.enable=true
value.converter.schemas.enable=true

# Kafka Connect 클러스터의 메타데이터 저장 토픽 설정, 운영은 replication.factor=3 권장
offset.storage.topic=connect-offsets
offset.storage.replication.factor=1

# 플러그인 참조 경로
plugin.path={{ plugin_path | join(',') }}
```
