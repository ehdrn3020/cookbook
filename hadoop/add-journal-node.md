# Journal Node 추가

> 작성일: 2026-07-16

- hdfs-site.xml에서 저널노드 경로 확인

```javascript
<property>
    <name>dfs.journalnode.edits.dir</name>
    <value>/tmp/hadoop/dfs/journalnode</value>
</property>
```

- 확인된 경로에 VERSION 파일 추가

```javascript
/tmp/hadoop/dfs/journalnode/myhdfs/current/VERSION
namespaceID=123123123
clusterID=CCCC-AAAA-BBB-4270-9393-asdfasdf
cTime=1638123123123
storageType=JOURNAL_NODE
layoutVersion=-10
```

- 저널 노드 실행
	- in_use.lock
		- 같은 JournalNode 저장 디렉터리를 두 프로세스가 동시에 사용하는 것을 막는 잠금 파일
	- edits.sync
		- 다른 저널노드에서 누락된 edit log를 내려받을 때 사용하는 임시 디렉터리
		- 동기화 완료된 이후 폴더 안의 파일들은 사라짐
