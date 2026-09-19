# HA 구성

> 작성일: 2025-10-28

- HiveServer2 : HiveServer2 HA = ZooKeeper 기반으로 HA 구성
	- hive-site.xml

```javascript
<property>
  <name>hive.server2.support.dynamic.service.discovery</name>
  <value>true</value>
</property>
<property>
  <name>hive.zookeeper.quorum</name>
  <value>zk1:2181,zk2:2181,zk3:2181</value>
</property>
<property>
  <name>hive.server2.zookeeper.namespace</name>
  <value>hiveserver2</value>
</property>
```

- MetaStore : 다중 Thrift URI 설정
	- hive-site.xml

```javascript
<property>
  <name>hive.metastore.uris</name>
  <value>thrift://server1:9083,thrift://server2:9083</value>
</property>
```
