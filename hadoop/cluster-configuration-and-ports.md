# Cluster 구성 및 사용포트

> 작성일: 2025-10-18

- Name Node
	- 8020 (RPC), 9870 (웹 UI)
- Data Node
	- 9866 (클라이언트/NN와 블록 송수신), 9867 (IPC / Metadata), 9864 (웹 UI)
- MapReduce JobHistory Server
	- MR 잡 수행 이력/로그 보는 서버 (Spark/YARN만 쓰면 필요없음)
	- 19888 (웹 UI)
- (YARN) Resource Manager
	- CPU/메모리 같은 자원 현황을 수집, 자원을 줄지 스케줄링/할당, 노드상태 모니터링
	- 8032 (IPC), 8088 (웹 UI)
- (YARN) Node Manager
	- 컨테이너(작업 단위)를 생성/실행/종료, RM에 주기적 보고
	- 8042 (웹 UI), 8040/8041 등(컨테이너 통신)
- Journal Node (QJM)
	- Active NN은 edit log를 과반수(quorum)의 JN에 기록, Standby NN는 JN에서 tail하여 최신 상태 유지
	- 8485 (IPC), 8480 (웹 UI)
- ZooKeeper - NN 호스트에 ZKFC 프로세스 띄우고, Active NN에 장애가 나면 다른 NN 자동 승격 조정
	- ZKFC (ZK Failover Controller) : 각 NN 호스트에서 돌아가는 프로세스, NN 로컬 IPC/HTTP 포트와 연계 (2181)
