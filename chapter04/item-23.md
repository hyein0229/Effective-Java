# Item 23. 태그 달린 클래스보다는 클래스 계층구조를 활용하라

## 태그 달린 클래스란?

두 가지 이상의 의미를 표현할 수 있으며, 그중 현재 표현하려는 의미를 태그 값으로 알려주는 클래스

클래스 계층구조보다 훨씬 나쁜 설계이다.

다음은 원과 사각형을 태그 값으로 사용하는 클래스 예제이다.

```java
public class Figure {
    enum Shape {RECTANGLE, CIRCLE}

    // 태그
    private final Shape shape;

    // 사각형일때만 사용
    private final double length;
    private final double width;

    // 원형일때만 사용
    private final double radius;

    public Figure(Shape shape, double radius) {
        this.shape = shape;
        this.length = 0;
        this.width = 0;
        this.radius = radius;
    }

    public Figure(Shape shape, double length, double width) {
        this.shape = shape;
        this.length = length;
        this.width = width;
        this.radius = 0;
    }

    public double area() {
        switch (shape) {
            case RECTANGLE:
                return length * width;
            case CIRCLE:
                return Math.PI * (radius * radius);
            default:
                throw new IllegalArgumentException();
        }
    }
}
```

### 태그 달린 클래스의 단점

---

- 열거 타입 선언, 태그 피드, switch 문 등 쓸데없는 코드가 많다.
- 여러 구현이 한 클래스에 혼합되어 있어 가독성도 나쁘다.
- 다른 의미를 위한 코드가 언제나 함께 하므로 메모리도 많이 사용한다.
- 필드들을 final 로 하여 불변으로 사용하려면 해당 의미에 쓰이지 않는 필드들까지 생성자에서 초기화해야 한다. 즉, 쓰지 않는 필드들을 초기화하는 불필요한 코드가 늘어난다.
- 컴파일러가 도와줄 수 있는 것이 별로 없다.
    - 엉뚱한 필드를 초기화해도 런타임에야 문제가 드러난다.
    - 새로운 의미를 추가할 때마다 switch 문에도 추가가 필요한데 빠뜨렸을 때도 런타임에야 알 수 있다.
- 인스턴스 타입만으로는 현재 나타내는 의미를 알 길이 전혀 없다.
- **즉, 태그 달린 클래스는 장황하고, 오류를 내기 쉽고, 비효율적이다.**

자바와 같은 객체 지향 언어는 타입 하나로 다양한 의미의 객체를 표현하는 수단인, 클래스의 계층구조를 활용하는 **서브타이핑(subtyping)**을 제공한다. **따라서 태그 달린 클래스는 클래스 계층구조를 어설프게 흉내낸 아류일 뿐이다.**

## 클래스 계층구조로 변환

- 가장 먼저 **계층구조의 루트가 될 추상 클래스를 정의**한다.
    - 태그 값에 따라 동작이 달라지는 메소드들을 루트 클래스의 추상 메소드로 선언한다.
    - 태그 값에 상관이 동작이 일정한 메소드들은 루트 클래스에 일반 메소드로 추가한다.
    - 모든 하위 클래스에서 공통 사용되는 데이터 필드들도 루트 클래스로 올린다.
- 루트 클래스를 확장하는 **구체 클래스를 정의**한다.
    - 각자의 의미에 해당하는 데이터 필드들, 메소드를 추가한다.
    - 루트 클래스의 추상 메소드를 오버라이딩하여 구현한다.

```java
public abstract class Figure {
    public abstract double area(); // 태그 값에 따라 달라지는 메소드
}

public class Circle extends Figure {
    private final double radius;

    public Circle(double radius) {
        this.radius = radius;
    }

    @Override
    public double area() {
        return Math.PI * (radius * radius);
    }
}

public class Rectangle extends Figure{
    private final double length; 
    private final double width;

    public Rectangle(double length, double width) {
        this.length = length;
        this.width = width;
    }

    @Override
    public double area() {
        return length * width;
    }
}
```

### 계층구조의 장점

---

- 다른 의미의 코드들은 남아있지 않는다.
- 자신과 관련 없는 데이터 필드들은 존재하지 않는다.
- 컴파일러가 생성자에서 final 필드들을 초기화하고 있는지, 추상 메소드를 구현했는지 체크해준다.
- switch 문에서 실수로 구현하지 않는 것때문에 런타임 오류가 발생하지도 않는다.
- 루트 클래스의 코드를 건드리지 않고도 독립적으로 계층구조를 확장할 수 있다.
    
```java
public class Square extends Figure{

    private final double side;

    public Square(double side) {
        this.side = side;
    }

    @Override
    public double area() {
        return side * side;
    }
}
```
    

## 요약 정리

- 태그 달린 클래스를 써야 하는 상황은 거의 없다.
- 태그 필드가 등장한다면 태그를 없애고 **클래스 계층구조**로 대체하는 방법을 생각하자.