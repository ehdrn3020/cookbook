# 툼스톤(tombstone)

> 작성일: 2025-11-15

- 값(value)이 null, / "이 key에 해당하는 데이터는 삭제"라는 신호
- 툼스톤 설정 예제( 토픽 메세지로 안보냄 )

```javascript
{
  "transforms": "unwrap",
  "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
  "transforms.unwrap.drop.tombstones": "true"  // ★ 포인트
  // delete.handling.mode는 기본값 none이라고 가정
}
```
