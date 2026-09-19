# RichSinkFunction

> 작성일: 2026-01-22

- 사용 목적
	- 직접 Sink 로직을 구현 ( Flink가 제공하는 `sinkTo` 사용 안함 )
	- 외부 시스템과 연결하여 Connection생성/재사용/Close
	- 리소스 생명주기
	- 병렬성(subtask) 기반 제어가 필요한 경우

```java
RichSinkFunction vs SinkFunction

SinkFunction의 실행 흐름
TaskManager 시작
└─ Sink 인스턴스 생성 (직렬화/역직렬화)
└─ invoke()   ← 레코드 1건
└─ invoke()
└─ invoke()
(끝났는지 모름)
Task 종료

RichSinkFunction의 실행 흐름
TaskManager 시작
 └─ Sink 인스턴스 생성 (직렬화/역직렬화)
 └─ open()         ← 여기서 커넥션 생성
     └─ invoke()   ← 레코드 1건
     └─ invoke()
     └─ invoke()
 └─ close()        ← 여기서 flush / close
Task 종료

-> 한 번 연결해서 오래 쓰며, subtask 마다 lifecycle hook 관리가 가능
```

- 예시

```java
public class MySink extends RichSinkFunction<Event> {

    private Connection connection;

    @Override
    public void open(Configuration parameters) throws Exception {
        connection = DriverManager.getConnection(...);
    }

    @Override
    public void invoke(Event event, Context context) throws Exception {
        // connection 재사용
    }

    @Override
    public void close() throws Exception {
        connection.close();
    }
}
```
