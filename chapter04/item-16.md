# Item 16. public 클래스에선 public 필드가 아닌 접근자 메소드를 사용하라.

## 접근자 메소드 방식

```java
class Point {
	public double x; // public 필드는 외부에서 접근해서 수정이 가능!
	public double y;
}
```

public 으로 외부에서 직접 접근할 수 있기 때문에 캡슐화의 이점을 제공하지 못한다. 

- API 를 수정하지 않고는 내부 표현 수정이 불가하다.
- 불변식을 보장할 수 없다.
- 필드 접근 시 부수 작업을 수행할 수 없다.

필드를 private 로 바꾸고 getter 메소드를 사용한다.

```java
class Point {
	private double x; // 필드는 priavte 로 선언
	private double y;
	
	public Point(double x, double y) {
		this.x = x;
		this.y = y;
	}

	public double getX() { return x; } // 접근자 (getter)
	public double getY() { return y; }

	public void setX(double x) { this.x = x; } // 변경자 (setter)
	public void setY(double y) { this.y = y; }
}
```

- 외부에 접근자를 제공함으로써 내부 표현은 언제나 바꿀 수 있는 유연성을 얻는다.
- 필드를 공개하면 이를 사용하는 클라이언트 코드가 있어 내부 표현을 마음대로 변경 불가하다.

**하지만, 예외적으로 package-private 클래스 혹은 private 중첩 클래스라면 필드를 노출해도 문제 없다.** 그 클래스가 표현하려는 추상 개념만 올바르게 표현해주면 된다. 오히려 클라이언트 코드 면에서나 접근자 방식보다 깔끔하다. **클라이언트 코드가 이 클래스 내부 표현에 묶이기는 하지만, 클라이언트도 어차피 해당 클래스를 포함하는 패키지 안에서만 동작하는 코드이므로 코드 변경 시에도 패키지 내로 국한되므로 괜찮다.**

public 클래스의 필드가 어차피 불변이라면 노출해도 괜찮지 않을까? 여전히 좋지 않다.

- 불변식은 보장할 수 있게 된다.
    - 인스턴스의 생성자에서 유효성 작업을 통해 불변식은 유지할 순 있다.
- 그러나, API 를 변경하지 않고는 내부 표현 방식을 바꿀 수 없다는 단점은 여전히 존재한다.
- 필드를 읽을 때의 부수 작업은 수행할 수 없다.

## 핵심 정리

즉, public 클래스의 가변 필드를 절대 직접 노출하도록 하면 안된다.

그러나, package-private 클래스나 private 중첩 클래스에선 필드를 노출하는 편이 나을 수 있다.