# __consumer_offsets 토픽

> 작성일: 2025-10-12

- 내부 토픽으로, 컨슈머 그룹이 어떤 토픽의 파티션을 어디까지(offset) 읽었는지 저장
- 주요 저장 정보 : group.id, topic, partition, commit offset
- 컨슈머가 재시작되거나 리밸런싱 되어도 해당 토픽을 통해 이전부터 다시 읽음
