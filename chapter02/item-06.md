# Item 06. 불필요한 객체 생성을 피하라

## 불필요한 객체 생성이란

*똑같은 기능의 객체를 매번 생성하기보다는 객체 하나를 재사용하는 편이 좋다.*

```java
 String s = new String("bikini"); // 1. 새로운 String 형 인스턴스 생성
 String s = "bikini"; // 2. 동일 인스턴스 재사용
```

new String("bikini") 에서 이미 생성자의 매개변수로 넘어가는 “bikini” 자체가 해당 생성자 호출로 만들어지는 String 형 인스턴스와 기능적으로 완전히 똑같다. 그에 반해, 아래는 하나의 String 인스턴스만을 사용한다. 이 방식을 사용하면 **JVM 안에서 이와 같은 문자열 리터럴을 사용하는 모든 코드는 같은 객체를 재사용함이 보장된다.**

생성자 대신 정적 팩토리 메소드 방식을 제공하면 이 정적 팩토리 메소드를 사용해서 불필요한 객체 생성을 제어할 수 있다. 객체 재사용은 불변 객체거나 혹은 꼭 불변 객체가 아니더라도 가변 객체 중 상태가 바뀌지 않을 것임이 분명하다면 가능하다.

___

**불필요한 객체 생성을 피해야 하는 경우**

1. 불변 객체, 혹은 가변 객체 중에 상태가 변하지 않는 객체
2. 생성 비용이 커서 캐싱하여 재사용할 수 있는 경우
3. 어댑터 (어댑터는 동작을 위이함 뒷단 객체의 인터페이스 역할이므로 뒷단 객체 하나당 하나씩만 필요)
4. 오토박싱으로 인한 인스턴스 생성을 피해라

## 불필요한 객체 생성을 피하자

### 생성 비용이 비싼 객체

*생성 비용이 큰 객체가 반복하여 필요할 때 캐싱을 할 수 있다면 캐싱하여 재사용해라*

```java
// 주어진 문자열이 로마 숫자인지 확인하는 메소드
static boolean isRomanNumeral(String s) {
		return s.matches("^(?=.)M*(C[MD]|D?C{0,3})"
		+ "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
}
```

```java
// matches 메소드 내부에서 Pattern 인스턴스의 생성
public boolean matches(String regex) {
    return Pattern.matches(regex, this);
}
```

여기서 문제는 String.matches 메소드이다. matches 메소드는 정규표현식으로 문자열 형태를 확인하는 가장 쉬운 방법이나, matches 메소드 내부에선 Pattern 인스턴스를 만들어 사용하므로 성능이 중요한 상황에선 반복하여 사용하기엔 적합하지 않다.

정규표현식용 Pattern 인스턴스는 한 번 쓰고 버려져 바로 GC의 대상이 된다. **Pattern 은 입력 받는 정규표현식에 해당하는 유한 상태 머신 (finite state machine) 을 만들기 때문에 인스턴스 생성 비용이 높다.**
💡

`유한 상태 머신 (finate state machine, FSM) 이란?`

상태와 상태 간의 전환을 기반으로 동작하는 동작 시스템이다. 즉, 어떤 상태에서 어떤 입력이 들어왔을 때 어디로 이동해야 하는 가를 나타내는 시스템인 것이다. 구성요소는 상태(state), 전환 조건 (Transition Condition), 동작 (Action)이 있다.

즉, 정규표현식 매칭은 복잡한 패턴 매칭이므로 FSM 으로 바꾸어 입력된 문자를 하나씩 읽으며 현재 상태에서 다음 상태로 이동하는 방식으로 일치 여부를 빠르게 구할 수 있다.

___

**Pattern.matches 내부 동작**

```java
public static boolean matches(String regex, CharSequence input) {
    Pattern p = Pattern.compile(regex); // 컴파일하여 Pattern 인스턴스 생성
    Matcher m = p.matcher(input); // matcher 생성, input 마다 가변이므로 재사용 불가
    return m.matches(); // input 과 정규표현식의 일치 여부 반환 
}
```

```java
public static Pattern compile(String regex) {
    return new Pattern(regex, 0); // 인스턴스 생성
}
```

컴파일하여 새로운 Pattern 인스턴스가 생성된다. 따라서 동일한 정규표현식을 가지고 String.matches 메소드가 빈번히 호출된다면 **컴파일한 Pattern 인스턴스를 미리 생성하여 캐싱해두고 재사용하는 것이 성능을 개선할 수 있다.**

```java
public class RomanNumerals {
    // 컴파일하여 Pattern 인스턴스를 미리 생성
		private static final Pattern ROMAN = Pattern.compile(
		"^(?=.)M*(C[MD]|D?C{0,3})"
		+ "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");

		static boolean isRomanNumeral(String s) {
				return ROMAN.matcher(s).matches(); // 생성된 Pattern 인스턴스 재사용
		}
}
```

- 장점
    - 성능을 개선할 수 있다.
    - 비즈니스 로직에 맞게 이름을 부여할 수 있어 코드의 의미도 더 명확해진다.

하지만 해당 메소드가 한 번도 호출되지 않는다면 ROMAN 필드가 쓸데없는 것 아닌가?

이러한 문제를 막기 위해서 메소드 최초 호출 시에 필드를 초기화하는 지연 초기화 (Lazy initialization) 방식을 사용할 수 있지만 굳이? 지연 초기화는 그만큼 로직이 들어가므로 코드가 복잡해지나 성능이 크게 향상되지 않을 때가 더 많다.

```java
public class RomanNumerals {
    private static class LazyHolder {
        static final Pattern ROMAN = Pattern.compile(
            "^(?=.)M*(C[MD]|D?C{0,3})"
            + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$"
        );
    }
		
		// 메소드 호출 시 LazyHolder 클래스가 로드되어 ROMAN 초기화됨
    static boolean isRomanNumeral(String s) {
        return LazyHolder.ROMAN.matcher(s).matches();
    }
}
```

---

### 어댑터 패턴 

중간 다리 역할

Map.keySet() → Map 객체의 키들을 담은 Set 뷰를 반환

---

### 오토박싱 (auto boxing)

*의도치 않은 오토박싱으로 무지막지하게 인스턴스가 생성되는 것을 피하자*

오토박싱이란 기본 타입과 박싱된 기본 타입(참조형)을 섞어 쓸 때 자동으로 상호 변환해주는 기술이다.

```java
private static long sum() {
		Long sum = 0L;
		for (long i = 0; i <= Integer.MAX_VALUE; i++)
			sum += i; // long -> Long 박싱하면서 인스턴스가 생성됨

		return sum;
} 
```

sum 변수를 Long 으로 선언하여 불필요한 Long 인스턴스가 엄청난게 생성되어 성능이 악화된다. Long 타입을 long 으로만 바꿔주어도 10배 이상 성능이 향상된다고 한다. 

따라서 박싱된 기본 타입보단 기본 타입을 사용해라


❓ **그러면 long → Long 박싱 시에는 무조건 인스턴스가 생성될까?**

컴파일러는 박싱 시 내부적으로 new Long 이 아닌 Long.valueOf 메소드를 호출하도록 한다. valueOf 메소드는 자바 공식 문서에 따르면 이미 생성된 값은 캐싱을 통해 재사용하여 최적화할 수 있도록 로직이 생성되어 있다고 한다. 따라서 항상 새로운 인스턴스가 생성되는 것은 아니고 이미 사용한 값은 캐싱된 값으로, 사용되지 않은 값에 대해선 new Long 으로 인스턴스를 생성하여 반환한다.

Returns a `Long` instance representing the specified `long` value. If a new `Long` instance is not required, this method should generally be used in preference to the constructor [`Long(long)`](https://docs.oracle.com/javase/8/docs/api/java/lang/Long.html#Long-long-), as this method is likely to yield significantly better space and time performance by caching frequently requested values. Note that unlike the [corresponding method](https://docs.oracle.com/javase/8/docs/api/java/lang/Integer.html#valueOf-int-) in the `Integer` class, this method is *not* required to cache values within a particular range.

## 하지만, 객체 생성을 피해야 한다로 오해하지 마라

요즘의 JVM은 작은 객체를 생성하고 회수하는 일이 크게 부담되지 않는다. 프로그램의 명확성, 간결성, 기능을 위해 객체를 추가 생성하는 것이라면 좋은 것이다. 의미가 없는 같은 객체를 불필요하게 생성하지 말라는 것이다.

아주 무거운 객체가 아니면 단순히 객체 생성을 피하기 위해 자체적으로 `객체 풀`을 만들지 마라. 일반적으로 자체 객체 풀은 코드를 헷갈리게 하고 메모리 사용량을 늘리고 성능을 떨어트려 객체 풀 자체가 큰 비용이 될 수 있다. 객체 풀이 잘 활용된 예는 **DB Connection Pool** 이 있다. 데이터베이스 연결 같은 경우는 생성 비용이 워낙 크기 때문에 풀을 만들고 재사용하는 것이 낫다.

요즘의 GC는 성능이 좋아 가볍고 작은 객체는 직접 만든 객체 풀보다 훨 빠르다.