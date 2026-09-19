# HA NameNode 승격

> 작성일: 2025-10-25

## 전제조건

- 자동 fail over 설정: `dfs.ha.automatic-failover.enabled=true`
- fencing 설정 : dfs.ha.fencing.methods=sshfence , dfs.ha.fencing.ssh.private-key-files=/myuser/.ssh/id_rsa
- QJM(JournalNode), ZKFC, Zookeeper 가동중

## 프로세스

1. zkfc는 zookeeper에 연동되어 nn01, nn02서버의 health check 실행
2. nn01의 8020이 내려가고 zkfc가 세션을 읽음
3. 우아한 전환(graceful failover) 시도 : nn02 zkfc → nn01 rpc(8020)로 transitionToActive 호출
4. 우아한 전환 실패 펜싱방법 실행 : 펜싱은 기존 Active 노드가 더 이상 메타데이터/서비스를 제공하지 못하도록 '물리적·논리적으로 차단'하는 절차
5. 펜싱을 통해 nn02 sshd → nn01 → nn01에서 점유 리소스 모두 해제
6. 펜싱 OK 후 nn02 ZKFC가 로컬 nn02에 transitionToActive 호출 → nn02 active로 승격
