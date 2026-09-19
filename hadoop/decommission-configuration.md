# Decommission 설정

> 작성일: 2026-04-24

- **dfs.exclude 파일에 제거할 노드 추가**

```plain text
1) hdfs-site.xml 수정 Active, Standby 둘 다 설정 필요
<property>    
  <name>dfs.hosts.exclude</name>    
  <value>/rnd/hadoop/default/etc/hadoop/conf/dfs.exclude</value>
</property>

2) 제외 호스트 작성
etc/hadoop/conf/dfs.exclude
>>>
server10
server12
```

- **NameNode에 반영 (`refreshNodes`)**

```plain text
/rnd/hadoop/default/bin/hdfs dfsadmin -refreshNodes
```

- **replication 완료될 때까지 대기**

```plain text
/rnd/hadoop/default/bin/hdfs dfsadmin -report
>>>
Name: 172.11.222.333:9866 (server42)
Hostname: server42
Decommission Status : Normal
....
| Decommission Status 상태  | 의미        |
| ------------------------ | --------     |
| Decommission in progress | 아직 복제 중  |
| Decommissioned           | 완료          |
| Normal                   | 아직 제외 안됨 |
```

- 상태 확인 및 이후 작업

```plain text
http://namenode:9870에서 확인 가능
watch -n 5 "/rnd/hadoop/default/bin/hdfs dfsadmin -report | grep -i 'Under replicated'"

# 데이터 노드 종료
/rnd/hadoop/default/bin/hdfs --daemon stop datanode
# 노드매니저 종료
/rnd/hadoop/default/bin/yarn --daemon stop nodemanager
```
