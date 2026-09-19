# GCD ( Greatest Common Divisor )

> 작성일: 2026-02-08

- GCD Codec은 숫자 컬럼의 "공통 분모(최대공약수)"를 찾아서 값을 더 작게 만든 뒤 압축 저장 코덱
- ClickHouse 23.9에 새로 추가

```javascript
Create Table Example ( ... 
	bid_v2 Decimal(11,5) CODEC(GCD, ZSTD)
	
예제 데이터
123.45000
123.46000

내부 Int 값:
12,345,000
12,346,000

이 값들의 GCD:
GCD = 1,000
```
