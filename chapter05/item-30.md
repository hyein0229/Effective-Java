# Item 30. 이왕이면 제네릭 메서드로 만들라

클래스와 마찬가지로 메서드도 제네릭으로 만들 수 있다.

매개변수화 타입(예. `List<String>`)을 받는 정적 유틸리티 메서드는 보통 제네릭이다. 대표적으로 Collections의 ‘알고리즘’ 메서드(binarySearch, sort 등)은 모두 제네릭이다.

```java
public static <T extends Comparable<? super T>> void sort(List<T> list) {
    list.sort(null);
}

public static <T> int binarySearch(List<? extends Comparable<? super T>> list, T key) {
    if (list instanceof RandomAccess || list.size()<BINARYSEARCH_THRESHOLD)
        return Collections.indexedBinarySearch(list, key);
    else
        return Collections.iteratorBinarySearch(list, key);
}
```

## 제네릭 메서드 작성법

### 두 집합의 합집합을 반환하는 union 메서드 - 로 타입 사용

```java
public static Set union(Set s1, Set s2) {
    Set result = new HashSet(s1);
    result.addAll(s2);
    return result;
}
```

- 컴파일은 되지만 두 개의 경고가 발생한다.
- new HashSet(s1) 과 result.addAll(s2) 부분에서 unchecked call warning 이 출력된다.
- 경고를 없애기 위해선 메서드를 타입 안전하게 만들어야 한다.
- 제네릭으로 만들자
- 메서드 선언에서 입력 2개, 반환 1개의 원소 타입을 타입 매개변수로 만든다.
    - **타입 매개변수 목록은 메서드의 제한자와 반환 타입 사이에 온다.**
- 그리고, 메서드 내부에서도 이 타입 매개변수만 사용하도록 수정한다.

```java
public static <E> Set<E> union(Set<E> s1, Set<E> s2) {
    Set<E> result = new HashSet<>(s1);
    result.addAll(s2);
    return result;
}
```

- 단순한 제네릭 메서드는 이 정도면 충분하다.
- 이 메서드는 경고 없이 컴파일되며, 타입 안전하고, 쓰기도 쉽다.
- 위 메서드는 입력 2개, 반환 1개의 타입이 모두 같아야 한다.
- 이를 한정적 와일드카드 타입을 사용하여 더 유연하게 개선할 수 있다.
    - 공통 조상을 가진 하위 타입까지 허용할 수 있게 된다.
    - s1, s2 가 E의 하위 타입이기만 하면 서로 달라도 된다.
    
    ```java
    public static <E> Set<E> union(Set<? extends E> s1, Set<? extends E> s2)
    ```
    

## 제네릭 싱글턴 메서드

- 때때로 불변 객체를 여러 타입으로 활용할 수 있게 만들어야 할 때가 있다.
- 제네릭은 런타임에 타입이 소거되므로 하나의 객체를 어떤 타입으로든 매개변수화할 수 있다.
- 이렇게 하려면 요청한 타입 매개변수에 맞게 매번 그 객체의 타입으로 바꿔주는 정적 팩터리를 만들어야 한다.
    - 이 패턴을 제네릭 싱글턴 팩터리라고 한다.
    - 하나의 불변 객체를 여러 제네릭 타입으로 재사용할 수 있도록 하는 패턴이다.
- 대표적인 예시로 Collections.reserveOrder 같은 함수 객체가 있다.
    - 타입별 Comparator를 반환하여 내림차순 정렬에 사용하도록 한다.
    - `ReverseComparator.REVERSE_ORDER` 가 불변객체가 된다.

```java
@SuppressWarnings("unchecked")
public static <T> Comparator<T> reverseOrder() {
    return (Comparator<T>) ReverseComparator.REVERSE_ORDER;
}
```

```java
Arrays.sort(integers, Collections.reverseOrder()); // Integer[] 정렬
Arrays.sort(strings, Collections.reverseOrder()); // String[] 정렬
```

### 제네릭 싱글턴 패턴 - 항등함수를 담은 클래스

- 자바 라이브러리의 Function.identity를 사용하면 되지만 직접 구현해본다.
- 항등함수 객체는 상태가 없으므로 요청마다 매번 객체를 생성하는 것은 낭비다. 재활용해야 한다.
- 제네릭이 실체화된다면 타입별로 하나씩 만들어야 하지만, 타입이 소거되므로 제네릭 싱글턴 하나면 된다.

```java
public class GenericSingletonPattern {
    private static UnaryOperator<Object> IDENTITY_FN = (t) -> t;
		
	// 비검사 형변환 경고를 숨긴다.
    @SuppressWarnings("unchecked")
    public static <T> UnaryOperator<T> identityFunction() {
        return (UnaryOperator<T>) IDENTITY_FN;
    }

    // 제네릭 싱글턴을 사용하는 예
    public static void main(String[] args) {
        String[] strings = { "삼베", "대마", "나일론" };
        UnaryOperator<String> sameString = identityFunction();
        for (String s : strings)
            System.out.println(sameString.apply(s));

        Number[] numbers = { 1, 2.0, 3L };
        UnaryOperator<Number> sameNumber = identityFunction();
        for (Number n : numbers)
            System.out.println(sameNumber.apply(n));
    }
}
```

- `UnaryOperator<T>`로 형변환하면서 비검사 형변환 경고가 발생한다.
- 하지만,  `UnaryOperator<T>`로 형변환하여도 타입 안전하다는 것을 검증할 수 있다.
    - 항등함수는 입력값을 수정 없이 그대로 반환하기 때문이다.
- 따라서, 비검사 형변환 경고를 @SuppressWarnings 어노테이션을 적용하여 숨긴다.

## 재귀적 타입 한정(recursive type bound)

- 드물지만, **자기 자신이 들어간 표현식**을 사용하여 타입 매개변수의 허용 범위를 한정할 수 있다.
- 즉, 타입 매개변수 자신을 타입 한정으로 사용하는 기법
- 재귀적 타입 한정은 주로 타입의 자연적 순서를 정하는 Comparable 인터페이스와 함께 쓰인다.

```java
public interface Comparable<T> {
    int compareTo(T o);
}
```

- 타입 매개변수 T는 `Comparable<T>`를 구현한 타입이 비교할 수 있는 원소의 타입을 정한다.
- 실제로 거의 모든 타입은 자신과 같은 타입의 원소와만 비교할 수 있다.
    - String은 `Comparable<String>`을 구현하고, Integer는 `Comparable<Integer>`를 구현한다.
- Comparable을 구현한 원소의 컬렉션을 입력받는 메서드들은 그 원소들을 주로 정렬, 검색, 최댓값/최솟값 구하는 식으로 사용된다.
- 따라서, 컬렉션에 담긴 모든 원소가 상호 비교될 수 있어야 한다.

```java
public static <E extends Comparable<E>> E max(Collection<E> c);
```

- 재귀적 타입 한정을 이용해 상호 비교할 수 있음을 표현한 메서드 선언이다.
    - `Comparable<E>`에 자기 자신 E가 다시 들어간 것이 재귀적 타입 한정의 핵심
- `<E extends Comparable<E>>`는 “모든 타입 E는 자신과 비교할 수 있다”라고 읽을 수 있다.

## 요약 정리

- 클라이언트에서 명시적 형번환해야 하는 메서드보다 제네릭 메서드가 더 안전하며 사용하기 쉽다.
- 제네릭 타입과 마찬가지로 메서드도 형변환 없이 사용하는 것이 좋다.
    - 많은 경우 그러기 위해선 제네릭 메서드가 되어야 한다.