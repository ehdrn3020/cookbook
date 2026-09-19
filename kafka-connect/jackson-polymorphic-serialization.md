# Jackson의 다형성 직렬화 기능

> 작성일: 2025-10-19

- 컬렉션이나 필드 타입이 인터페이스/추상클래스일 때 JSON의 한 항목을 어떤 실제 클래스로 읽을지 맵핑 함
- @JsonTypeInfo + @JsonSubTypes
