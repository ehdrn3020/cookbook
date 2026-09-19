# CF 수에 따른 저장 장단점

> 작성일: 2025-11-23

## CF 분리할 때 장점

- Write 병렬 처리
- 디스크 I/O 효율 : 전용 CF로 빼면, scan/get 시 해당 CF 파일만 접근해 효율적
- 정책 분리 : CF마다 VERSIONS, TTL, 압축 방식 등을 따로 줄 수 있음
- 도메인 가독성 : 역할 별로 나누면, 나중에 보는 사람도 이해하기 쉬움

## CF가 분리할 때 단점

- MemStore 수 증가 : CF가 늘수록 Region당 Store가 많아져 flush/compaction 관리 오버헤드가 증가 함
- MemStore, HFile, Compaction 등 관리 오버헤드 증가
- RegionServer 입장에서 I/O & 메모리 관리가 더 복잡해짐
