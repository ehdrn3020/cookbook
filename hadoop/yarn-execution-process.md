# Yarn 실행 프로세스

> 작성일: 2026-04-09

```javascript
[YARN / MapReduce 실행 관계]

1) 제출
Client
  └─ sqoop / hive / spark-submit / jar 실행
        │ job 제출
        ▼
2) 자원 총괄
ResourceManager (RM)
  └─ 클러스터 전체 자원 관리
  └─ 어느 NodeManager에 AM을 띄울지 결정
        │ AM용 container 할당
        ▼
3) 작업 관리자
ApplicationMaster (AM)
  └─ "이 job의 총괄 관리자"
  └─ map 몇 개 필요한지 계산
  └─ RM에 task container 요청
  └─ task 상태 모니터링 / 재시도 / 완료처리
        │ map/reduce container 요청
        ▼
4) 실제 실행 노드들
NodeManager (NM)
  ├─ NM on dn09
  │    └─ AM container 실행 가능
  │
  ├─ NM on dn02
  │    └─ Map task container 실행
  │
  └─ NM on dn17
       └─ Map task container 실행


[YARN / Spark 실행 관계]

Client
  ↓
ResourceManager
  ↓
ApplicationMaster (AM)
  ↓
Driver (SparkContext) --> AM = Driver 같은 노드일 경우가 많
  ↓
Executors (여러 노드)
  ↓
Tasks
```
