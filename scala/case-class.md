# CaseClass

> 작성일: 2025-12-27

- Scala에서 DTO / Record / Schema 역할
- 사용 이유 - 타입 안정성, 스키마명확, Dataset API 활용, IDE에서 자동완성, 그리고 분산 환경에서 안전한 불변 객체를 제공

```javascript
case class User(user_id: Int, name: String)
val userDs: Dataset[User] = spark.read
  .option("header", "true")
  .option("inferSchema", "true")
  .csv("/user/user.csv")
  .as[User]
```
