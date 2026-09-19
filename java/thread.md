# 쓰레드

> 작성일: 2025-12-20

- 구현과 실행

```javascript
class MyThread extends Thread { public void run() { ..작업내용 ..} }
MyThread t1 = new MyThread();
t1.start(); // 실행
```

- 우선순위 조정 가능 : th1.setPriority(5); | 기본값 5
- 데몬쓰레드 : 보조적 역할 | 일반쓰레드가 모두 종료되면 자동종료 | garbage collector, 화면갱신 등 사용
- sleep(), interrupt(), suspend(), resume(), join(), yield() | wait(), notify()
- public synchronized void calcSum() 을 통해 임계영역(lock을 하여 1개의 쓰레드만 접근) 설정
