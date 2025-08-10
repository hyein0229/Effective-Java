# Item 26. 로 타입은 사용하지 말라
## 제네릭 클래스, 제네릭 인터페이스

- 통틀어 **제네릭 타입(generic type)** 이라고 한다.
- 클래스와 인터페이스 선언에 타입 매개변수(type parameter)가 쓰인 경우
- 예를 들어 List 인터페이스는 원소의 타입을 나타대는 타입 매개변수 E를 받는다. → List<E>
- 각각의 제네릭 타입은 일련의 **매개변수화 타입(parameterized type)** 을 정의한다.
    - 클래스(인터페이스) 이름 뒤 꺽쇠갈호 안에 실제 타입 매개변수들을 나열한다.
    - `List<String>` 은 원소 타입이 String인 리스트를 뜻하는 매개변수화 타입이다.
    - String은 정규 타입 매개변수 E에 해당하는 실제 타입 매개변수이다.
- 제네릭 타입을 하나 정의하면 그에 딸린 **로 타입(raw type)** 도 함께 정의된다.
    - 로 타입이란? 제네릭 타입에서 타입 매개변수를 전혀 사용하지 않을 때를 말한다.
    - `List<E>`의 로 타입은 List 이다.

제네릭 지원 전에 컬렉션 로 타입을 사용한 경우

```java
class RawCollection {
    private final Collection stamps = null; // 로 타입 사용

    void sample() {
        stamps.add(new Coin()); // "unchecked call" 경고를 내뱉는다.
    }

    private static class Coin {
        // ...
    }
}
```

```java
void sample() {
    for (Iterator i = stamps.iterator(); i.hasNext(); ) {
        Stamp stamp = (Stamp) i.next(); // ClassCastException을 던진다.
        stamp.cancel();
    }
}
```

- stamp 컬렉션을 의도했지만 Coin 인스턴스를 넣어도 컴파일 오류는 나타나지 않는다.
- unchecked call 경고 메세지를 보여주지만 오류는 나지 않는다.
- stamps 에서 원소를 꺼내서 Stamp 타입으로 형변환 시 ClassCastException 을 던진다.
    - 즉, **컴파일 때가 아닌 런타임에야 오류를 발견할 수 있다.**
    - 오류는 가능한 즉시, 이상적으로는 컴파일 때 발견하는 것이 좋다.

매개변수화된 컬렉션 타입 - 타입 안전성을 확보한다.

```java
class ParameterizedCollectionSample {
    private final Collection<Stamp> stamps = null; // 매개변수화된 타입 사용
}
```

- 컴파일러가 stamps 에는 Stamp 인스턴스만 넣어야 함을 인지할 수 있다.
- 따라서 엉뚱한 타입의 인스턴스를 넣으려고 한다면 컴파일 오류가 발생하여 무엇이 잘못됐는지 알려준다.
    - `error: incompatible types: Coin cannot be converted to Stamp`
- 컴파일러는 컬렉션에서 원소를 꺼내는 곳에 보이지 않는 형변환을 추가하여 절대 실패하지 않음을 보장한다.
- **로 타입을 쓰면 제네릭이 안겨주는 안전성과 표현력을 모두 잃게 된다.**
    - 그렇다면 로 타입을 왜 만들어놓은 것일까? → 호환성 때문
    - 제네릭 없이 짜여진 기존 코드를 모두 수용하면서 제네릭을 사용하기 위해선 만들어야 했다.

## `List` vs `List<Object>`

- List 와 같은 로 타입은 안되지만, `List<Object>`처럼 임의 객체를 받는 매개변수화 타입은 괜찮다.
- List 는 제네릭 타입에서 완전히 발을 뺀 것이고, `List<Object>`는 모든 타입을 허용한다는 의미를 컴파일러에게 전달한 것이다.
- **매개변수화 타입을 사용하면 로 타입을 사용할 때와 달리 타입 안전성을 잃는다.**
    - 매개변수로 List를 받는 메소드에 `List<String>`을 넘길 수 있지만, `List<Object>`일 때는 불가능하다.
    - `List<String>`은 로 타입인 List 의 하위이지만,  `List<Object>`의 하위 타입은 아니다.

런타임에 실패하는 예 - 로 타입을 매개변수로 받는 경우

```java
class FailWithRawType {
    public static void main(String[] args) {
        List<String> strings = new ArrayList<>();
        unsafeAdd(strings, Integer.valueOf(42));
        String s = strings.get(0); // 컴파일러가 자동 형변환 코드를 넣는다.
    }

    // 1. 로 타입 사용
    private static void unsafeAdd(List list, Object o) {
        list.add(o); // unchecked call 경고 메시지 출력, 정상 컴파일
    }

    // 2. 매개변수화 타입 사용
    private static void unsafeAdd(List<Object> list, Object o) { // 컴파일 오류!!!
        list.add(o);
    }
}
```
- 로 타입 사용의 경우
    - list.add 부분에서 unchecked call 경고 메시지가 출력되지만 정상 컴파일된다.
    - strings.get(0)에서 Integer → String 변환 시 ClassCastException 발생.
- 하지만 `List<Object>`로 매개변수 타입을 바꾸면 컴파일 오류가 발생
    - 즉, `List<String>`을 `List<Object>`로 바꿀 수 없어 incompatible types 오류가 난다.

    
## 타입을 모르고 사용하고 싶을 때 - 비한정적 와일드카드 타입

로 타입은 원소의 타입을 몰라도 되므로 사용하고 싶어질 수 있다. 다음은 두 집합을 받아 서로 공통된 것들의 개수를 반환하는 메소드이다. 로 타입은 안전하지 않다.

```java
class UseUnknownElementRawTypeSample {
    static int numElementsInCommon(Set s1, Set s2) {
        int result = 0;
        for (Object o1 : s1) {
            if (s2.contains(o1)) {
                result++;
            }
        }
        return result;
    }
}
```

### 비한정적 와일드 카드타입(unbounded wildcard type)

```java
class UseUnknownElementRawTypeSample {
		// Set<String>, Set<Integer> 모두 받을 수 있음
    static int numElementsInCommon(Set<?> s1, Set<?> s2) {
	    ...
    }
}
```

- **제네릭 타입을 쓰고 싶지만 실제 타입 매개변수가 무엇인지 신경 쓰고 싶지 않은 경우 사용한다.**
- 타입에 물음표(?)를 사용한다. List<E> → List<?>
- 어떤 타입이든 담을 수 있는 가장 범용적인 매개변수화 타입이다.
- 타입 안전하고 유연성을 제공한다.

### 로 타입 vs 비한정적 와일드카드 타입

- 와일드카드 타입은 안전하고, 로 타입은 안전하지 않다.
- 로 타입 컬렉션에는 아무 원소나 넣을 수 있어 타입 불변식을 훼손하기 쉽다. (unsafeAdd 처럼)
- 반면, **Collection<?>에는 (null 외에는) 어떤 원소도 넣을 수 없다.**
    - 즉, 컬렉션의 타입 불변식을 훼손하지 못하게 막았다.
    - null 외의 다른 원소를 넣으려고 하면 컴파일 오류가 발생한다.
    - 그럼 언제 사용하나? 여러 타입을 받아 **read-only** 로만 처리하면 되는 작업일 때

## 로 타입을 사용하는 예외

- class 리터럴에는 로 타입을 사용한다.
    - 자바 명세는 class 리터럴에 매개변수화 타입을 사용하지 못하게 했다 (배열과 기본 타입은 허용한다).
    - List.class, String[].class, int.class
    - `List<String>.class`, `List<?>.class` → 매개변수화 타입은 사용 불가
- instanceof 연산자는 비한정적 와일드카드 타입 이외의 매개변수화 타입에는 적용할 수 없다.
    - 로 타입이든 비한정적 와일드카드 타입이든 동작은 같다.
    - 비한정적 와일드카드 타입의 꺾쇠괄호와 물음표는 아무런 역할 없이 코드만 지저분하게 하므로, 차라리 로 타입을 사용하는 것이 깔끔하다.
    
    ```java
    if (o instanceof Set) {
        Set<?> s = (Set<?>) o;
        // ...
    }
    ```
    

## 요약 정리

- **로 타입 사용은 런타임 오류가 발생할 수 있어 사용하면 안된다.**
- 로 타입은 기존 제네릭 도입 전의 코드와 호환성을 위해 제공될 뿐이다.
- `Set<Object>`는 어떤 타입의 객체도 담을 수 있고, `Set<?>` 은 어떤 타입 객체도 담을 수 없다.

| 매개변수화 타입 | parameterized type | `List<String>` |
| --- | --- | --- |
| 실제 타입 매개변수 | actual type parameter | `String` |
| 제네릭 타입 | generic type | `List<E>` |
| 정규 타입 매개변수 | formal type parameter | `E` |
| 비한정적 와일드카드 타입 | unbounded wildcard typ | `List<?>` |
| 로 타입 | raw type | `List` |
| 한정적 타입 매개변수 | bounded type parameter | `<E extends Number>` |
| 재귀적 타입 한정 | recursive type bound | `<T extends Comparable<T>>` |
| 한정적 와일드카드 타입 | bounded wildcard type | `List<? extends Number>` |
| 제네릭 메서드 | generic method | `static <E> List<E> asList(E[] a)` |
| 타입 토큰 | type token | `String.class` |