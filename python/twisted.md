# Twisted

> 작성일: 2025-12-02

- Twisted = Python의 이벤트 기반 네트워크 프레임워크(비슷 asyncio, Node.js)
- 소켓, TCP, HTTP, timer 같은 걸 논블로킹 이벤트 루프로 처리하는 라이브러리
- 예제

```javascript
from twisted.internet import reactor
...
self.time_out = reactor.callLater(60.0, self.do_loop)
...
reactor.addSystemEventTrigger("before", "shutdown", work.shutdown)
reactor.run()
```
