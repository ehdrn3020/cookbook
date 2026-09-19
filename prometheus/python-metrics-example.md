# Python metrix 전송 예제

> 작성일: 2026-01-10

```python
from prometheus_client import start_http_server, Gauge

MONITOR_UP = Gauge(
    "znode_monitor_up",
    "ZNode monitor daemon is running"
)

if __name__ == "__main__":
    # Prometheus metrics endpoint
    start_http_server(HOST_PORT)
    MONITOR_UP.set(1)
```
