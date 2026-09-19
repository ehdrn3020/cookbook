# KeyedProcessFunction

> 작성일: 2026-01-18

- keyBy 된 스트림에서, "상태(State) + 타이머(Time)"를 사용할 수 있는 이벤트 처리 함수
- `processElement()`

```python
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
# 반드시 override
@Override
# keyedProcessFunction< Key타입, Input타입, Output타입 >
public class FraudDetector extends KeyedProcessFunction<Long, Transaction, Alert>
  # 각 거래(Transaction) 이벤트마다 한 번씩 호출
	public void processElement(
		Transaction transaction, # 입력 값
		Context context,  # "이벤트가 어떤 key, 어떤 시간 기준으로 처리되는지" 알려주는 핸들러
		Collector<Alert> collector # 출력 
	)
```

- DataStream#keyBy를 사용해 스트림을 파티셔닝
	- key 값 기준으로 논리적으로 partition ( 단지 routing 규칙 생성 됨 )
- `open()`
	- 연산자(Task) 인스턴스가 시작될 때 딱 한 번 호출된다.
	- State 초기화, 외부 리소스(DB, client) 생성 같은 준비 작업을 수행한다.
	- 이벤트 처리 로직은 절대 넣으면 안 되는 초기화 단계다.
- `onTimer()`
	- 등록된 타이머 시간이 도래했을 때 Flink가 자동으로 호출한다.
	- key별 시간 기반 로직(만료, timeout, cleanup) 을 처리하는 용도다.
	- processElement와 같은 thread에서 실행되며, 해당 key에 대해서만 동작한다.
- `cleanUp()` *(사용자 정의 메서드)*
	- 상태(State)와 타이머를 명시적으로 정리하기 위한 헬퍼 메서드다.
	- 등록된 타이머를 취소하고 관련 state를 초기화한다.
