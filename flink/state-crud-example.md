# State CRUD Example

> 작성일: 2026-08-25

## state 저장

```javascript
private transient ValueState<Boolean> flagState;
private transient MapState<String, Integer> countState;

@Override
public void open(OpenContext openContext) {
    // 1. ValueState 정의
    ValueStateDescriptor<Boolean> flagDescriptor =
            new ValueStateDescriptor<>(
                    "flag-state",
                    Types.BOOLEAN
            );
    flagState = getRuntimeContext().getState(flagDescriptor);

    // 2. MapState 정의
    MapStateDescriptor<String, Integer> countDescriptor =
            new MapStateDescriptor<>(
                    "count-state",
                    Types.STRING,
                    Types.INT
            );
    countState = getRuntimeContext().getMapState(countDescriptor);
}
```

## state 사용

```javascript
public void processElement(Event event, Context ctx, Collector<Event> out) 
	throws Exception {

    // ValueState
    Boolean flag = flagState.value();   // 조회
    flagState.update(true);             // 저장/수정
    flagState.clear();                  // 삭제

    // MapState
    Integer chatCount = countState.get("chat");  // 조회
    countState.put("chat", 10);                 // 저장/수정
    countState.put("view", 20);
    countState.remove("chat");                   // 특정 항목 삭제
    countState.clear();                          // 전체 Map 삭제
}

// 내부 상태
userA
 ├─ ValueState
 │    flag = true
 │
 └─ MapState
      chat = 10
      view = 20
userB
 ├─ ValueState
 │    flag = false
 │
 └─ MapState
      chat = 30
      view = 50
```
