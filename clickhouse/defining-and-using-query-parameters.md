# Defining and Using Query Parameters

> 작성일: 2026-07-16

- 쿼리 파라미터를 세션에 설정하는 방식
	- 반복 사용 가능
	- 세션이 달라지면 초기화 됨

```javascript
SET param_a = 13;
SET param_b = 'str';
SET param_c = '2022-08-04 18:30:53';
SET param_d = {'10': [11, 12], '13': [14, 15]};

SELECT
   {a: UInt32},
   {b: String},
   {c: DateTime},
   {d: Map(String, Array(UInt8))};

13    str    2022-08-04 18:30:53    {'10':[11,12],'13':[14,15]}
```
