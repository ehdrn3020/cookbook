# URL에서 Import, 파일로 Export

> 작성일: 2026-02-13

```javascript
SELECT *
FROM url('https://storage.example.com/metrics/000.parquet')
LIMIT 1
SETTINGS max_http_get_redirects = 1; // redirect 허용

INSERT INTO FUNCTION file(
    '/tmp/metrics_{_partition_id}.csv.gz',
    'CSV',
    'gzip'
)
PARTITION BY toDate(ts)
SELECT * FROM metrics;

>>>>> 생성 결과
/tmp/metrics_2026-02-13.csv.gz
```
