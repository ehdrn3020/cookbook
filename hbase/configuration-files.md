# 설정파일

> 작성일: 2025-11-05

- backup-masters : 액티브 HMaster가 다운되면 스탠바이 HMaster 후보 목록
- hbase-env.cmd : 윈도우 전용
- hbase-env.sh : 리눅스에서 HBase 데몬 기동 시 적용되는 환경 변수와 JVM 옵션.
- hbase-policy.xml : RPC/접근 정책(보안 레벨별 서비스 허용 제어)
- **hbase-site.xml** : HBase의 핵심 동작 파라미터 설정 파일
- log4j.properties: HMaster/RegionServer/클라이언트의 로깅 레벨·출력 경로 설정.
- regionservers : RegionServer를 띄울 호스트 목록.
