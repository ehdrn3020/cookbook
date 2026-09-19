# this() : 생성자에서 다른 생성자 호출할 때 사용

> 작성일: 2025-11-17

- 호출 시 첫 줄에서만 가능, 참조변수 this 사용

```javascript
class Car2 {
	Car2(){
		this("white","auto",4);
	}
	...
	Car2(String color, String gearType, int door){
		this.color = color ...
	}
}
```

Java는 단일 상속만 허용, 대신에 포함관계 또는 interface로 같은 기능 구현
