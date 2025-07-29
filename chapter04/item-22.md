# Item 22. 인터페이스는 타입을 정의하는 용도로만 사용하라
**인터페이스의 용도**

- 자신을 구현한 클래스의 인스턴스를 참조할 수 있는 타입 역할
- 즉, **자신의 인스턴스로 무엇을 할 수 있는지**를 클라이언트에게 얘기해주는 것

## 상수 인터페이스(안티패턴)

위 지침에 맞지 않는 예로 상수 인터페이스가 존재한다. 상수 인터페이스란 메소드 없이, 상수를 뜻하는 static final 필드로만 가득 찬 인터페이스를 말한다. 해당 상수들을 사용하려는 클래스에선 정규화된 이름을 쓰는 것을 피하고자 인터페이스를 구현하곤 한다.

```java
public interface PhysicalConstants {
	// Avogadro's number (1/mol)
	static final double AVOGADROS_NUMBER = 6.022_140_857e23;
	// Boltzmann constant (J/K)
	static final double BOLTZMANN_CONSTANT = 1.380_648_52e-23;
	// Mass of the electron (kg)
	static final double ELECTRON_MASS = 9.109_383_56e-31;
}
```

**정규화된 이름이 무엇이냐**

자바에서 상수를 사용할 때 그 상수가 속한 클래스나 인터페이스 이름까지 명시하는 방식이다.

상수 인터페이스를 사용한다면? 상수 이름만으로 간단히 사용이 가능하다.

```java
public interface Constants {
    int ERROR_CODE = 404;
}

int code = Constants.ERROR_CODE; // 정규화된 이름 사용
```

```java
public class MyClass implements Constants {
    public void printCode() {
        System.out.println(ERROR_CODE); // 클래스 이름 없이 직접 사용 가능
    }
}
```

---

### 상수 인터페이스의 단점

- 상수는 내부 구현이다. 따라서 상수 인터페이스 구현은 내부 구현을 클래스의 API로 노출하는 행위이다.
    - 내부 구현이므로 클라이언트에겐 아무런 의미가 없는데 그러한 기능을 제공하는 것처럼 보여진다.
    - 심하게는 클라이언트 코드가 내부 구현인 이 상수들에 종속될 수 있다.
- 다음 릴리즈에서 위 상수들을 안쓰더라도 바이너리 호환성을 위해 여전히 인터페이스를 구현하고 있어야 한다.
- final 이 아닌 클래스가 상수 인터페이스를 구현한다면 모든 하위에서의 이름공간이 정의된 상수로 오염된다.
    - 즉, 사용하지 않는 상수도 가지게 된다.

___

### 해결방법
- 특정 클래스나 인터페이스와 강하게 연관된 상수라면 그 클래스나 인터페이스 자체에 추가해야 한다.
    - 대표적으로 Integer 에 선언된 MIN_VALUE, MAX_VALUE 상수가 있다.
- 열거 타입으로 나타내기 적합한 상수라면 열거 타입으로 만들어 공개한다.
- 그것도 아니라면, 인스턴스화할 수 없는 유틸리티 클래스에 담아 공개한다.
    - 유틸리티 클래스에 정의된 상수 사용을 위해선 기본적으로 클래스 이름까지 함께 명시해야 한다.
    - 빈번히 사용 시엔 정적 임포트(static import)하여 생략도 가능하다.
    
    ```java
    package com.effectivejava.science;

    public class PhysicalConstants { // 유틸리티 클래스
    
    	private PhysicalConstants() { } // 외부에서 인스턴스화하는 것을 방지한다.
    
    	public static final double AVOGADROS_NUMBER = 6.022_140_857e23;
    	public static final double BOLTZMANN_CONST = 1.380_648_52e-23;
    	public static final double ELECTRON_MASS = 9.109_383_56e-31;
    }
    
    // 자바 7부터 지원하는 숫자 리터럴에 사용한 밑줄(_)에 주목해보자. 
    // 밑줄은 숫자 리터럴 값에는 영향을 주지 않으면서 가독성을 높인다.
    // 5자리 이상이라면 밑줄을 사용하는 것을 고려해보자. 
    // 십진수 리터럴도 (정수든 부동소수점이든) 밑줄을 사용해 세 자릿씩 묶어주는 것이 좋다.
    ```
    
## 요약 정리

- 인터페이스는 타입을 정의하는 용도로만 사용해야 한다.
- 상수 공개용 수단으로 사용하지 말자.
- 상수는 연관된 클래스나 인터페이스에만 정의하거나, 또는 Enum, 유틸리티 클래스에 담아 공개한다.