# Item 03. 생성자나 열거 타입으로 싱글턴임을 보증하라
**싱글턴(singleton)이란?**

인스턴스를 오직 하나만 생성할 수 있는 클래스를 말한다. 싱글턴의 예로는 무상태(stateless)객체나 설계상 유일해야 하는 시스템 컴포넌트를 들 수 있다. 

하지만, 클래스를 싱글턴으로 만들면 클라이언트가 테스트하기 어려워질 수 있다. 싱글턴 인스턴스를 가짜(mock) 구현으로 대체할 수 없기 때문이다.

싱글턴을 구현할 때 공통되는 점은 **private 으로 생성자를 감추어 외부에서 생성하는 것을 막는 것**이다.


**여기서 무상태(stateless)란?**

**내부에 변수가 없는 객체의 상태**를 말한다. Spring 에서 Bean 은 외부에서 주입받는 Bean 을 제외하고 변수가 존재하지 않기 때문에 무상태 객체로 볼 수 있다.


싱글턴을 만드는 방식은 보통 3가지 중 하나이다.

- public static final 필드 방식
- 정적 팩토리 방식
- 열거 타입 방식

## public static final 필드 방식의 싱글턴

*public static final 멤버에서 인스턴스를 한번만 초기화한 후 사용한다.*

```java
public class Elvis {
		public static final Elvis INSTANCE = new Elvis();
		private Elvis() { ... }
		
		public void leaveTheBuilding() {
				...
		}
}
```

- private 생성자는 public static final 필드를 초기화할 때 한번만 호출된다.
- 이 방식의 큰 장점은 해당 클래스가 싱글턴임이 API에 명백히 드러난다는 것이다.
    - final 필드이므로 절대로 다른 객체를 참조할 수가 없다.
- 매우 간결하다.

## 정적 팩토리 메소드 방식

```java
public class Elvis {
		private static final Elvis INSTANCE = new Elvis(); // private 선언
		private Elvis() { ... }
		
		// 인스턴스를 반환하는 정적 팩토리 메소드
		// 항상 같은 객체를 반환한다.
		public static Elvis getInstance() {
				return INSTANCE;
		}

		public void leaveTheBuilding() {
				...
		}
}
```

- 첫번째 장점은 API를 바꾸지 않고도 나중에 싱글턴에서 다른 방식으로 변경이 가능하다는 점이다.
- 두번째 장점은 정적 팩토리를 제네릭 싱글턴 팩토리로 만들 수 있다는 점이다.
- 세번째 장점은 해당 메소드를 Supplier<Elvis> 에 메소드 레퍼런스로 넘겨줄 수 있다는 것이다. 

제네릭 싱글턴 팩토리

```java
public class IdentityFunction {
    // 제네릭 싱글턴 인스턴스
    private static final UnaryOperator<Object> IDENTITY = t -> t;

    @SuppressWarnings("unchecked")
    public static <T> UnaryOperator<T> identityFunction() {
        return (UnaryOperator<T>) IDENTITY;
    }
}

// String, Integer 등 다양한 타입으로 생성 가능
UnaryOperator<String> strOp = IdentityFunction.identityFunction();
UnaryOperator<Integer> intOp = IdentityFunction.identityFunction();
strOp.apply("hello")
```

위 두 가지 방식으로 만든 싱글턴 객체를 직렬화하려면 단순히 Serializable 을 구현하는 것으론 모자라다. 모든 인스턴스의 필드를 일시적(transient, 직렬화 시 제외)이라고 선언하고 readResolve 메소드를 제공해야 한다. 이렇게 하지 않으면 직렬화된 인스턴스를 역직렬화할 때마다 새로운 인스턴스가 만들어진다.

```java
private Object readResolve() {
		// '진짜' Elvis를 반환하고 오버라이드 한 역직렬화 과정에서 생겨난 Elvis는 Garbage Collector에 맡긴다
		return INSTANCE;
}
```

## 열거 타입 방식 - 바람직한 방식

원소가 하나인 열거 타입을 선언한다. 싱글턴을 만들기에 가장 안전한 방법

**Enum 은 JVM 수준에서 클래스 로딩 시에 단 하나의 인스턴스만 생성되도록 보장된다.**

```java
public enum Elvis {
		INSTANCE;
		
		public void leaveTheBuliding() {
			...
		}
}

// 인스턴스 사용
Elvis elvis = Elvis.INSTANCE
```

- public static final 필드 방식과 비슷하나 더 간결하고 추가 노력 없이 직렬화 가능하다.
    - readResolve 구현 없이도 역직렬화 시에 싱글턴이 깨지지 않는다.
- 대부분 상황에서 원소가 하나뿐인 enum 타입을 만드는 방식이 가장 바람직하다.