# Metrics Reporting

> 작성일: 2026-07-05

- Iceberg가 테이블을 읽고/쓰는 과정에서 내부적으로 무슨 일이 있었는지 관측하기 위해 쓰는 기능
	- 몇 개 data file을 scan / pruning / scan planning에 얼마나 걸렸는지
	- commit에 얼마나 걸렸는지, commit retry가 있었는지
	- 추가/삭제된 data file 수가 몇 개인지
- Iceberg 1.1.0부터 `MetricsReporter`, `MetricsReport` API를 지원
- Type of Reports
	- ScanReport   = 읽기/스캔 계획 수립 메트릭
	- CommitReport = 쓰기/커밋 메트릭
- Available Metrics Reporters
	- Metrics Reporter는 위의 `ScanReport`, `CommitReport`를 어디로 보낼지 정하는 구현체
		- LoggingMetricsReporter : 로그 파일에 출력하는 방식
		- RESTMetricsReporter : REST endpoint로 보내는 방식
		- Custom MetricsReporter : 직접 Reporter 만드는 방식
	- LoggingMetricsReporter 예시 출력

```javascript
INFO org.apache.iceberg.metrics.LoggingMetricsReporter - Received metrics report: 
ScanReport{
    tableName=scan-planning-with-eq-and-pos-delete-files, 
    snapshotId=2, 
    filter=ref(name="data") == "(hash-27fa7cc0)", 
    schemaId=0, 
    projectedFieldIds=[1, 2], 
    projectedFieldNames=[id, data], 
    scanMetrics=ScanMetricsResult{
        totalPlanningDuration=TimerResult{timeUnit=NANOSECONDS, totalDuration=PT0.026569404S, count=1}, 
        resultDataFiles=CounterResult{unit=COUNT, value=1}, 
        resultDeleteFiles=CounterResult{unit=COUNT, value=2}, 
        totalDataManifests=CounterResult{unit=COUNT, value=1}, 
        totalDeleteManifests=CounterResult{unit=COUNT, value=1}, 
        scannedDataManifests=CounterResult{unit=COUNT, value=1}, 
        skippedDataManifests=CounterResult{unit=COUNT, value=0}, 
        totalFileSizeInBytes=CounterResult{unit=BYTES, value=10}, 
        totalDeleteFileSizeInBytes=CounterResult{unit=BYTES, value=20}, 
        skippedDataFiles=CounterResult{unit=COUNT, value=0}, 
        skippedDeleteFiles=CounterResult{unit=COUNT, value=0}, 
        scannedDeleteManifests=CounterResult{unit=COUNT, value=1}, 
        skippedDeleteManifests=CounterResult{unit=COUNT, value=0}, 
        indexedDeleteFiles=CounterResult{unit=COUNT, value=2}, 
        equalityDeleteFiles=CounterResult{unit=COUNT, value=1}, 
        positionalDeleteFiles=CounterResult{unit=COUNT, value=1}}, 
    metadata={
        iceberg-version=Apache Iceberg 1.4.0-SNAPSHOT (commit 4868d2823004c8c256a50ea7c25cff94314cc135)}}
```
