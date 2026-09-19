# ClickStack

> 작성일: 2026-03-09

- ClickStack = ClickHouse + Observability(HyperDX) 도구 묶음
- 기존 Observability 복잡해서 모든 데이터를 ClickHouse 하나에 저장 ( logs, traces, metrics, session )

| 항목 | ELK | ClickStack |
| --- | --- | --- |
| Storage | Elasticsearch | ClickHouse |
| UI | Kibana | HyperDX |
| Collector | Logstash | OTEL / Vector |
| 성능 | 중간 | 매우 빠름 |
| 비용 | 높음 | 낮음 |
