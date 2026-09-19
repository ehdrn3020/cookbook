# Yarn External Shuffle Service

> 작성일: 2025-11-03

- Spark가 YARN 위에서 돌 때, Executor가 내려가도(동적 할당·오류 등) 그 Executor가 남겨둔 Shuffle 파일을 NodeManager 측 데몬이 대신 보관·서빙해 주는 서비스
- YARN의 NodeManager aux-service로 띄워지며 Spark는 이를 External Shuffle Service라고 부릅니다.

## 왜 필요한가?

- 동적 할당 필수 요소: `spark.dynamicAllocation.enabled=true` 인 경우, executor를 제거해도 이전 Stage의 shuffle 결과(중간파일)를 유지하여 다음 Stage/Job에서 재사용
- 신뢰성: 실패로 executor가 죽어도 shuffle 데이터를 노드 단위로 계속 서비스 → 재시작 비용/재계산 감소.
- 자원 효율: executor 수를 공격적으로 줄여도(스케일-인) shuffle 유지를 보장

## 동작 방식

- 각 NodeManager에서 **Spark YARN Shuffle Service**(클래스: `org.apache.spark.network.yarn.YarnShuffleService`)가 **7337/tcp** 같은 포트로 Listen
- executor가 종료되더라도 **해당 노드의 로컬 디스크**에 남은 shuffle 파일을 대신 제공
- Executor/Driver ↔ ESS 사이에 Netty 기반으로 블록을 조회

## 설정

- yarn-site.xml

```javascript
<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>{{ yarn_nm_aux_services }}</value>
</property>
<property>
    <name>yarn.nodemanager.aux-services.spark_shuffle.class</name>
    <value>{{ yarn_nm_spark_shuffle_class }}</value>
</property>
<property>
    <name>yarn.nodemanager.aux-services.spark_shuffle.port</name>
    <value>7337</value>
</property>
```

- spark-3.2.0-yarn-shuffle.jar : 모든 Node Manager 설치 서버에 적용

```javascript
# 제공 파일
/rnd/spark/default/yarn/spark-3.2.0-yarn-shuffle.jar

# Yarn 경로로 복사
/rnd/hadoop/default/share/hadoop/yarn
```
