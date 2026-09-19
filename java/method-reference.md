# 메서드 참조

> 작성일: 2025-12-20

- 하나의 메서드만 호출하는 람다식은 '메서드 참조'로 간단히

```javascript
Integer method(Stirng s) { return Integer.parseInt(s); }
-> Function<String, Integer> f = (String s) -> Integer.parseInt(s);
-> Function<String, Integer> f = Integer::parseInt;
```
