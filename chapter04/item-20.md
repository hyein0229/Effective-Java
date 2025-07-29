# Item 20. 추상 클래스보단 인터페이스를 우선하라

자바가 제공하는 다중 구현 메커니즘은 인터페이스와 추상 클래스가 있다.

하지만, 추상 클래스와 인터페이스의 가장 큰 차이는 아래와 같다.

- **추상 클래스:** 구현 클래스는 반드시 추상 클래스 타입의 하위 클래스가 된다.
- **인터페이스:** 정의된 메소드를 구현만 하면 어떤 클래스를 상속하든 같은 타입으로 취급된다.

자바는 단일 상속만 지원하므로 추상 클래스 방식은 커다란 제약을 가진다. 

그러나 인터페이스는 여러 인터페이스를 implements 하여 구현만 하면 해당 타입들을 얻을 수 있다.

## 인터페이스의 장점
### 기존 클래스에도 쉽게 새로운 인터페이스를 구현해 넣을 수 있다.

- 인터페이스를 implements 하고 메소드 추가하고 구현만 하면 된다.
- java 버전이 올라가며 Comparable, Iterable, AutoCloseable 인터페이스가 새롭게 추가 되었을 때 표준 라이브러리의 기존 클래스가 이 인터페이스를 구현한 채로 릴리즈됐다.
- 추상 클래스로는 어렵다.
    - 두 클래스가 A 라는 추상 클래스를 상속받아 구현하면 A는 공통 조상이 된다.
    - 따라서, 클래스 계층 구조에 큰 혼란을 준다.

___

### 믹스인(mixin) 정의에 안성맞춤이다.
- **믹스인이란?**
    - 원래의 ‘주된 타입’ 외에도 특정 선택적 기능을 혼합하는 것
    - 즉, 기능을 쉽게 넣을 수 있다.
    - 예를 들어 Comparable 은 자신을 구현한 클래스의 인스턴스들끼리 순서를 정할 수 있는 기능을 제공
- 추상 클래스와 같은 계층 구조는 믹스인을 삽입하기 어렵다.

___

### 계층구조가 없는 타입 프레임워크를 만들 수 있다.

타입 계층구조는 수많은 개념을 구조적으로 잘 표현할 수 있지만, 현실에서 계층을 엄격히 구분하기 어려운 개념도 있다. 예를 들어 가수(singer) 인터페이스와 작곡가(songwriter) 인터페이스가 있다고 하자.

```java
public interface Singer {
	AudioClip sing(Song s);
}

public interface Songwriter {
	Song compose(int chartPosition);
}
```

현실에선 작곡을 하는 가수도 있으므로 아래와 같이 제3 인터페이스 싱어송라이터를 정의할 수 있다.

```java
public interface SingerSongwriter extends Singer, Songwriter { // 다중 구현
	AudioClip strum(); // 새로운 메소드까지 추가
	void actSensitive();
}
```

**클래스로 만든다면?**

- 다중 상속 불가
- 필요한 조합 전부를 각각의 클래스로 정의한 고도비만 계층구조가 만들어질 것이다.
    - 속성이 n 개라면 가능한 모든 조합의 수 2*n 개가 된다.
    - 조합 폭발 (combinato-rial explosion) 현상이다.

## 인터페이스의 default 메소드

- java 8부터 default 메소드로 인스턴스 메소드를 구현 형태로 제공 가능하다.
- 인터페이스의 메소드 중 구현 방법이 명백한 것이 있다면, 디폴트 메소드로 제공하여 일감을 덜 수 있다.
- 상속하려는 사람을 위해 @implSpec 자바독 태그를 붙여 문서화해야 한다.
- **제약 사항**
    - equals 와 hashcode 같은 Objects 메소드는 디폴트로 제공하지 마라
    - 인스턴스 필드를 가질 수 없고 public 이 아닌 정적 멤버도 가질 수 없다. (private 정적 메소드는 예외)
        - 즉, 디폴트만으로 구현 형태를 모두 제공할 수 없을 수 있다.

## 추상 골격 구현 클래스 (skeletal implementation)

- 인터페이스와 추상 클래스의 장점을 둘 다 취할 수 있는 방법이다.
- 인터페이스로는 타입을 정의하고, 필요하면 일부 디폴트 메소드도 제공한다. 그리고 골격 구현 클래스는 나머지 메소드들까지 구현한다. 즉, **인터페이스 구현 시 공통된 부분을 골격 구현 클래스로 해결한다.**
- 골격 구현은 인터페이스를 구현하려는 프로그래머의 일을 상당히 덜어준다는 장점이 있다.
- 디자인 패턴으로는 **템플릿 메소드 패턴**이라고 한다.
- 관례상 인터페이스명이 *Interface* 면 골격 구현 클래스는 *AbstractInterface* 로 짓는다.
    - java - AbstractCollection, AbstractList, AbstractSet 등

```java
// Concrete implementation built atop skeletal implementation
static List<Integer> intArrayAsList(int[] a) {
    Objects.requireNonNull(a);

    // Java 9 이상에서는 <> (다이아몬드 연산자) 사용 가능
    // Java 8 이하는 <Integer> 명시 필요
    return new AbstractList<>() {
        @Override
        public Integer get(int i) {
            return a[i]; // Autoboxing (int → Integer)
        }

        @Override
        public Integer set(int i, Integer val) {
            int oldVal = a[i];
            a[i] = val; // Auto-unboxing (Integer → int)
            return oldVal; // Autoboxing
        }

        @Override
        public int size() {
            return a.length;
        }
    };
}
```

예제를 보며 이해한다.

```java
// 인터페이스
public interface Phone {
  void booting();
  void greeting();
  void shutdown();
  void process();
}
```

```java
// 추상골격 구현 클래스 (보통 Abstract~의 네이밍을 사용한다)
public abstract class AbstractPhone implements Phone {

	// 같은 동작을 하는 메소드를 여기에 정의한다.
  @Override
  public void booting() {
    System.out.println("booting ...");
  }

  @Override
  public void shutdown() {
    System.out.println("shut down ...");
  }

  @Override
  public void process() {
    booting();
    greeting();
    shutdown();
  }
}
```

```java
public class IPhone extends AbstractPhone implements Phone {
	
	// 기종별 다른 행위는 Phone 을 implements 하여 구현
  @Override
  public void greeting() {
    System.out.println("I am iPhone");
  }
}
```

즉, 인터페이스를 클래스에서 구현할 때 공통된 행위를 하는 메소드들이 있다면 **추상 골격 구현 클래스를 통해 공통된 행위는 정의하고, 나머지 공통된 부분이 아닌 메소드에 대해서만 인터페이스를 implements 하여 구현하면 된다.** 이는 코드의 재사용도 높이고 프로그래머의 일을 덜어줄 수 있다.

---

### 시뮬레이트한 다중 상속

만약, 구조상 추상 골격 구현 클래스를 상속받아 확장하지 못하는 상황이라면 **해당 골격 구현을 확장한 클래스를 private 내부 필드로 두고, 각 메소드 호출 시 내부 클래스의 인스턴스에 전달하는 것이다.** 래퍼 클래스 방식과 비슷한 방식이다. 

```java
public class PhoneManufacturer {
  public void printManuFacturer() {
    System.out.println("Made by Apple");
  }
}
```

```java
// 골격 구현을 확장한 클래스
public class InnerAbstractPhone extends AbstractPhone {

  @Override
  public void greeting() {
    System.out.println("I am iPhone");
  }
}
```

```java
// 이미 상속을 받고 있으므로 다중 상속은 불가
public class IPhone extends PhoneManufacturer implements Phone {
  InnerAbstractPhone innerAbstractPhone = new InnerAbstractPhone(); // 내부 필드

  @Override
  public void booting() {
    innerAbstractPhone.booting(); // 내부에서 참조하고 있는 인스턴스의 메소드 호출
  }

  @Override
  public void greeting() {
    innerAbstractPhone.greeting();
  }

  @Override
  public void shutdown() {
    innerAbstractPhone.shutdown();
  }

  @Override
  public void process() {
    printManuFacturer();
    innerAbstractPhone.process();
  }
}
```

---

### 골격 구현 작성 방법

- 인터페이스를 살펴 다른 메소드들의 구현에 사용되는 기반 메소드들을 선정한다.
    - 즉, get() 과 size() 만 있으면 isEmpty() 작성이 가능하다.
- 골격 구현에서 이 기반 메소드들은 abstract 메소드가 된다. → 즉, 하위에서 무조건 구현해야 함
- 기반 메소드를 사용해 직접 구현할 수 있는 메소드를 모두 디폴트 메소드로 제공한다.
- 만약, 인터페이스의 메소드 모두가 기반 메소드와 디폴트 메소드가 된다면 골격 구현 클래스를 별도로 만들 이유는 없다. 그러지 못한 나머지 메소드들이 존재한다면 골격 구현 클래스를 만들어 남은 메소드들을 작성해 넣는다. (골격 구현 클래스에는 필요하면 public 이 아닌 필드와 메소드를 추가해도 된다.)

## 요약 정리
- 다중 구현용 타입으로는 인터페이스가 가장 적합하다.
- 복잡한 인터페이스라면 수고를 덜어주는 **골격 구현**을 함께 제공하는 방식을 고려하라.
- 골격 구현은 ‘가능한 한’ 인터페이스의 디폴트 메소드로 제공하여 그 인터페이스를 구현한 모든 곳에서 활용하도록 하는 것이 좋다.
    - 인터페이스 제약 때문에 골격 구현을 디폴트 메소드보단 추상 클래스로 (추상 골격 구현 클래스) 로 제공하는 경우가 더 흔하다.