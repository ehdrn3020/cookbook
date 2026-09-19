# DataNode 종료 방법

> 작성일: 2026-07-08

- **rnd/hadoop/default/bin/hdfs --daemon stop datanode**
	- 이건 해당 서버 로컬에서 DataNode 프로세스를 직접 종료하는 방식입니다.
	- 이 서버의 DataNode 프로세스를 그냥 stop 스크립트로 내림
- **/rnd/hadoop/default/bin/hdfs dfsadmin -shutdownDatanode datanode01:9867 upgrade**
	- NameNode를 통해 DataNode에 shutdown 요청을 보내는 방식
	- NameNode에게 "저 DataNode를 rolling upgrade/점검 모드로 정상 종료시켜라"라고 요청
	- upgrade 옵션을 주면 재기동 후 이전 upgrade 상태와 연계해서 처리 가능
		- upgrade : DataNode는 곧 재시작될 예정이니 잠깐 기다려라" 동작하게 하고, **fast start-up mode**가 활성화 됨. 재시작이 제시간에 안 되면 client는 timeout 후 해당 DataNode를 무시
