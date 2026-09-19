# HDFS Block 전송 동시성 튜닝

> 작성일: 2026-05-28

## NameNode Replication 관련 설정 튜닝

- block 복제 이동 속도 관련이라 decommission / rebalancing / replication / recovery 전부 영향
- namenode의 hdfs-site.xml 파일 (active + standby 포함)

```javascript
/* DataNode 1대당 동시에 수행 가능한 replication 수 (기본 값 : 2) */
<property>
  <name>dfs.namenode.replication.max-streams</name>
  <value>40</value>
  <description>하나의 DataNode에 대해 NameNode가 동시에 할당할 수 있는 복제 작업 수의 제한 값</description>
</property>

/* max-streams의 절대 상한 (기본 값 : 4, hard-limit = max-streams * 2 미만) */
<property>
  <name>dfs.namenode.replication.max-streams-hard-limit</name>
  <value>70</value>
  <description>하나의 DataNode에 대해 NameNode가 동시에 할당할 수 있는 복제 작업 수의 상한 값</description>
</property>

/* 한 번 scheduler loop에서 얼마나 많은 replication 작업을 생성할지 작업속도 결정 (기본 값 : 2)
DataNode 50대 × multiplier 10 = 약 500개
*/
<property>
  <name>dfs.namenode.replication.work.multiplier.per.iteration</name>
  <value>10</value>
  <description>NameNode가 한 번의 복제 스케줄링 반복에서 처리할 복제 작업량을 DataNode 수 기준 배수로 조절</description>
</property>
```

- 적용 후 설정 확인

```javascript
// reconfig 명령을 통해 namenode 재실행하지 않고도 설정 적용가능한 옵션 확인
/rnd/hadoop/default/bin/hdfs dfsadmin -reconfig namenode namenode01:8020 properties

// 적용 후 확인 ( active + standy 둘다 )
/rnd/hadoop/default/bin/hdfs dfsadmin -reconfig namenode namenode01:8020 start
/rnd/hadoop/default/bin/hdfs dfsadmin -reconfig namenode namenode01:8020 status

// 변경 값 확인
/rnd/hadoop/default/bin/hdfs getconf -confKey dfs.namenode.replication.max-streams
/rnd/hadoop/default/bin/hdfs getconf -confKey dfs.namenode.replication.max-streams-hard-limit
/rnd/hadoop/default/bin/hdfs getconf -confKey dfs.namenode.replication.work.multiplier.per.iteration
```

- 적용 후 리소스 비교

```javascript
// 속도 측정
hdfs dfsadmin -report | grep -i "Under replicated"
// 네트워크 사용량 (DataNode)
sar -n DEV 1
// RX Drop 확인 (DataNode)
netstat -i
// DISK I/O (DataNode)
iostat -x 1
// CPU NameNode
top -p $(pgrep -f NameNode)
```

## DataNode 요청 동시성 튜닝

- datanode의 /rnd/hadoop/default/etc/hadoop/hdfs-site.xml

```javascript
/* DataNode가 NameNode나 HDFS client로부터 받는 RPC 요청을 처리하는 handler thread 수
- block 정보 조회
- block report 관련 처리
- heartbeat 관련 응답 처리
- block 생성/삭제/복구 관련 제어 요청
- client/DataNode 간 메타성 RPC 요청
*/
<property>
  <name>dfs.datanode.handler.count</name>
  <value>20</value>
  <description>DataNode RPC 요청을 처리하는 handler thread 수</description>
</property>

/* DataNode가 block 데이터를 실제로 주고받는 전송 작업의 최대 동시 처리 수
- HDFS client가 DataNode에서 block 읽기 / 쓰기
- DataNode 간 block replication
- balancer / mover / decommission 중 block 이동
- pipeline write 중 downstream DataNode로 block 전달
*/
<property>
  <name>dfs.datanode.max.transfer.threads</name>
  <value>8192</value>
  <description>DataNode의 block read/write/replication 전송 처리 최대 transfer thread 수</description>
</property>
```
