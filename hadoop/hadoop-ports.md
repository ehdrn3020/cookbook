# Hadoop Port

> 작성일: 2026-03-30

| 구분 | From | To | Port | 필수 | 설명 |
| --- | --- | --- | --- | --- | --- |
| HDFS | NameNode | DataNode | 9866 | O | DataNode IPC / 제어 (Heartbeat 응답, 명령) |
| HDFS | NameNode | DataNode | 9867 | O | DN 상태 확인 및 내부 통신 |
| HDFS | DataNode | DataNode | 9867 | O | Block 복제 / Rebalance / Recovery |
| HDFS | DataNode | NameNode | 8020 | O | HDFS RPC (DN 등록, Heartbeat, BlockReport) |
| HDFS | DataNode | RDB Host | 3306 | O | Sqoop 사용 Host |
| HDFS | Client / DataNode | DataNode | 9867 | O | HDFS 데이터 Read / Write |
| YARN | ResourceManager | NodeManager | 8040 | O | Container 관리 RPC |
| YARN | NodeManager | ResourceManager | 8032 | O | NM 등록 + NM heartbeat |
| YARN | NodeManager | NameNode | 8020 | O | HDFS RPC |
| YARN | NodeManager | DataNode | 9867 | O | Block read/write |
| UI | 로컬 PC | DataNode | 9864 | 선택 | DataNode Web UI (운영/점검용) |
| UI | 로컬 PC | NodeManager | 8042 | 선택 | NodeManager Web UI |
| UI | 로컬 PC | ResourceManager | 8088 | 선택 | ResourceManager Web UI |
