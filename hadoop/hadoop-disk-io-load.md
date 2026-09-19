# Hadoop Disk I/O 부하

> 작성일: 2026-06-11

확인

```javascript
-- 가장 많이 I/O 사용하는 프로세스 확인
sudo iotop -oPa

-- 할당이 많이되는 프로세스 PID 확인
ps -ef | grep "du -sk /Data"
k3        3127 14326  0 16:16 ?        00:00:10 du -sk /Data2/dfs/dn/current/BP-1054196674-[운영IP]-1638254800248
k3       31057 14326  1 14:50 ?        00:01:20 du -sk /Data1/dfs/dn/current/BP-1054196674-[운영IP]-1638254800248
k3       32451 14326  1 16:35 ?        00:00:00 du -sk /Data3/dfs/dn/current/BP-1054196674-[운영IP]-1638254800248
k3       32560 32418  0 16:35 pts/0    00:00:00 grep --color=auto du -sk /Data

-- 실행 시간 확인
ps -o pid,etime,cmd -p 3127,31057,32451
 3127       19:45 du -sk /Data2/dfs/dn/current/BP-1054196674-[운영IP]-1638254800248
31057    01:45:04 du -sk /Data1/dfs/dn/current/BP-1054196674-[운영IP]-1638254800248
32451       00:30 du -sk /Data3/dfs/dn/current/BP-1054196674-[운영IP]-1638254800248
```

**Disk Usage cache refresh**

- 주기적으로 `du` 방식으로 디스크 사용량을 계산
- `fs.du.interval`은 Hadoop이 디스크 사용량을 다시 계산하는 주기
	- HDFS DataNode는 자신이 가진 블록 디렉터리의 사용량을 추적이 필요
- `fs.getspaceused.className` 통해 디스크 사용량을 계산하는 변경 ( du → df )
	- Hadoop 3.x 이상
- core-site.xml

```javascript
<property>
  <name>fs.getspaceused.classname</name>
  <value>org.apache.hadoop.fs.DFCachingGetSpaceUsed</value>
  <description>DataNode directory scanning using a DF based cached method</description>
</property>

<property>
  <name>fs.du.interval</name>
  <value>1800000</value>
  <description>Disk usage cache refresh every 30 minutes. default 10 minutes</description>
</property>
```

**DirectoryScanner**

- 데이터 노드 블록들을 검사 후 네임노드가 알고 있는 정보화 비교 및 동기화
- 진행 프로세스

```javascript
1) /Data1, /Data2, /Data3 아래 block/meta 파일 스캔
 - getdents64      # 디렉터리 목록 읽기
 - stat/newfstatat # 파일 존재, 크기, mtime 확인
 - openat          # 파일 열기
 - read            # 일부 메타데이터 읽기
2) DataNode 메모리 replica map과 비교
3) missing block file, missing metadata file, missing blocks in memory 확인
4) DataNode 내부 replica 상태 reconcile
5) 이후 block report / incremental report를 통해 NameNode에 반영
6) NameNode가 필요하면 replication 또는 invalidation/delete 지시
```

- hdfs-site.xml

```javascript
  <property>
    <name>dfs.datanode.directoryscan.interval</name>
    <value>172800</value>
    <description>Interval in seconds for DataNode DirectoryScanner. default 6 hours (48 hours)</description>
  </property>

  <property>
    <name>dfs.datanode.directoryscan.throttle.limit.ms.per.sec</name>
    <value>50</value>
    <description>Limits the DataNode DirectoryScanner scan time per second. default 1000</description>
  </property>

  <property>
    <name>dfs.datanode.directoryscan.threads</name>
    <value>1</value>
    <description>DirectoryScanner thread count</description>
  </property>
```
