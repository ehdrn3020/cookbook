# Node Rebalancing 옵션

> 작성일: 2026-01-14

## 옵션 값
### threshold
- 리밸런싱 종료 기준
- 평균 50% 노드에 데이터가 저장되어있을 때 threshold=10이면 허용 범위 45% ~ 55%

### Ddfs.balancer.bandwidthPerSec
- DataNode 한 대당 Balancer로 사용할 수 있는 최대 네트워크 대역폭
- 단위: bytes/sec, 50 MB/s
### setBalancerBandwidth 
- hdfs hdfs dfsadmin -setBalancerBandwidth 52428800
- 설정안하면 hdfs-site.xml 파일의 property = dfs.datanode.balance.bandwidthPerSec 따름
- 기본값 10MB

## HDFS Balancer Bandwidth Flow
```
      [ Balancer Process ]
      ---------------------
      |  요청 속도 제한   |
      |  -Ddfs.balancer.bandwidthPerSec
      ---------------------
                 |
                 |  (block move 요청)
                 v
      [ Network Transfer ]
                 |
                 v
      [ DataNode ]
      ---------------------
      | 실제 처리 속도 제한 |
      | dfs.datanode.balance.bandwidthPerSec
      | or setBalancerBandwidth
      ---------------------
                 |
                 v
           [ Disk I/O ]
```

## 실행
- setBalancerBandwidth 데이터노드의 속도 상한
- Ddfs.balancer.bandwidthPerSec는 balancer 프로세스의 속도 상한
- 둘 중 낮은 쪽이 실제 속도가 됨
```
./hdfs dfsadmin -setBalancerBandwidth 52428800   (50mb)
./hdfs balancer -Ddfs.balancer.bandwidthPerSec=50m -threshold 10
```

## 상태 확인
```
./hdfs balancer -status
```
