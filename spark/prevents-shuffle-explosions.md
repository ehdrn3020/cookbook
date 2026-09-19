# Prevents Shuffle Explosions

> 작성일: 2026-03-25

## broadcast join

- Spark에서 일반 join은 보통 양쪽 데이터를 같은 key 기준으로 재분배(shuffle)
- 작은 테이블은 굳이 shuffle 하지 않고 각 executor에 복사해두고 큰 테이블
- 예시

```javascript
# 일반 join
# - `df_orders` : 주문 데이터, 매우 큼
# - `df_customers` : 고객 정보, 상대적으로 작음
df_orders.join(df_customers,"customer_id")

# 브로드캐스트 조인
# - `df_customers` 를 executor들에게 미리 뿌린다
#- `df_orders` 는 원래 partition 상태를 유지한 채 join 가능

frompyspark.sql.functionsimportbroadcast
optimized=df_orders.join(broadcast(df_customers),"customer_id")
```

## Salting

- 양쪽이 크고 특정 key 쏠림이 심하면(**skewed key)** salting으로 heavy key를 분산
- 예시

```javascript
frompyspark.sql.functionsimportrand,concat_ws,col

SALT_BUCKETS=10
orders_salted=df_orders.withColumn(
	"salted_customer_id",
	concat_ws("_",col("customer_id"), (rand()*SALT_BUCKETS).cast("int"))
)
>>>
이 코드는 예를 들어 `customer_id=123` 을
- `123_0`
- `123_1`
- `123_2`
- ...
- `123_9`
즉 원래 한 key에 몰리던 주문 데이터를 10개 버킷으로 흩뿌린다.
```
