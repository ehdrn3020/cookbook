# Node Rebalancing 옵션

> 작성일: 2026-01-14

- threshold
	- 리밸런싱 종료 기준
	- 평균 50% 노드에 데이터가 저장되어있을 때 threshold=10이면 허용 범위 45% ~ 55%
- Ddfs.balancer.bandwidthPerSec
	- DataNode 한 대당 Balancer로 사용할 수 있는 최대 네트워크 대역폭
	- 단위: bytes/sec, 50 MB/s

```python
hdfs balancer \
  -threshold 10 \
  -Ddfs.balancer.bandwidthPerSec=50m

# 상태 확인
hdfs balancer -status
```
