# Item 31. 한정적 와일드카드를 사용해 API 유연성을 높이라

## 매개변수화 타입은 불공변이다.
- 불공변(invariant) 이다.
- 즉, `List<type1>`과 `List<type2>`는 서로 하위 타입도 상위 타입도 아니다.
- `List<Object>`에는 어떤 객체도 넣을 수 있으나 `List<String>`은 문자열만 넣을 수 있다.
    - 리스코프 치환 원칙에 어긋난다.
    - 부모 객체를 서브 객체로 대체했을 때 부모의 행위를 의도대로 할 수 없기 때문
- 따라서, 때론 불공변 방식보다 유연한 무언가가 필요하다. 이것이 **한정적 와일드카드 타입**이다.

## 한정적 와일드카드 타입
### 생산자(producer) 와일드카드 적용

- 와일드카드 타입을 사용하지 않은 pushAll 메소드
    
    ```java
    public void pushAll(Iterable<E> src) { // 여기서 src 가 생산자 역할
        for (E e : src) {
            push(E);
        }
    }
    ...
    public static void main(String[] args) {
        Stack<Number> numberStack = new Stack<>();
        Iterable<Integer> integers = ...;
        numberStack.pushAll(integers); // 불공변 타입으로 런타임 에러
    }
    ```
    
    - 정상 컴파일은 되나 런타임 오류 가능성이 있다.
    - Iterable src 의 원소 타입이 스택의 원소 타입과 일치한다면 정상 동작한다.
    - 그러나 `Stack<Number>`인 경우 pushAll(integers) 호출하는 경우
        - Integer는 Number의 하위 타입, `Iterable<Integer>`는 `Iterable<Number>`의 하위 타입 x
        - 즉, incompatible types 에러가 발생한다.
- **해결: 한정적 와일드카드 타입 사용**
- E 생산자(producer) 매개변수에 와일드카드 타입 적용
    
    ```java
    public void pushAll(Iterable<? extends E> src) {
        for (E e : src) {
            push(E);
        }
    }
    ```
    
    - 정상 컴파일되며 타입 안전하다.
    - `E`의 `Iterable`이 아니라 **`E`의 하위 타입의 `Iterable`**이어야 한다는 의미를 갖는다.

### 소비자(consumer) 와일드카드 적용

- Stack 안의 모든 원소를 입력 받은 컬렉션으로 옮겨 담는다.
- 와일드카드 타입을 사용하지 않은 popAll 메소드
    
```java
public void popAll(Collection<E> dst) {
    while (!isEmpty()) {
        dst.add(pop());
    }
}
...
public static void main(String[] args) {
    Stack<Number> numberStack = new Stack<>();
    Collection<Object> objects = ...;
    numberStack.popAll(objects); // imcompatible types 에러 발생!!!
}
```
    
- 주어진 컬렉션의 원소 타입이 스택의 원소 타입과 같다면 문제 없이 작동할 것이다.
- 하지만, 입력을 `Collection<Object>` 타입을 준다면?
    - `Collection<Number>`와 `Collection<Object>`는 호환 불가한 타입으로 에러 발생한다.
- **해결: 한정적 와일드카드 타입 사용**
- E 소비자(consumer) 매개변수에 와일드카드 타입 적용
    
```java
public void popAll(Collection<? super E> dst) {
    while (!isEmpty()) {
        dst.add(pop());
    }
}
```

- E의 Collection이 아니라 ‘E의 상위 타입’의 Collection이어야 한다.
- **유연성을 극대화하려면 원소의 생산자나 소비자용 입력 매개변수에 와일드카드 타입을 사용하라**

### 펙스(PECS) - 와일드카드 사용 공식

> producer-extends, consumer-super
>

- 매개변수화 타입 T가 생산자라면 <? extends T>를 사용해라
- 매개변수화 타입 T가 소비자라면 <? super T>를 사용해라
- 만약, 입력 매개변수가 생산자와 소비자 역할을 같이 한다면 와일드카드 타입을 써도 좋을 것이 없다.
- 생산자인 union 메소드에 펙스 적용
    
```java
public static <E> Set<E> union(Set<E> s1, Set<E> s2) // 적용 전, 후
public static <E> Set<E> union(Set<? extends E> s1, Set<? extends E> s2) 
```
    
- 이때 **반환 타입은 여전히 `Set<E>`** 임에 주의해야 한다.
    - 반환 타입에는 한정적 와일드카드 타입을 사용하면 안된다.
    - 반환 타입에 적용하면 클라이언트 코드에서도 와일드카드 타입을 사용해서 받아야 한다.
- 펙스 적용 후 클라이언트 코드
    
```java
Set<Integer> integers = Set.of(1, 3, 5);
Set<Double> doubles = Set.of(2.0, 4.0, 6.0);
Set<Number> numbers = union(integers, doubles);
```

- 와일드카드를 제대로 사용한다면 클라이언트는 와일드카드 타입이 사용된지도 모른다.
- **만약, 클래스 사용자가 와일드카드 타입을 신경 써야 한다면 API에 문제가 있을 가능성이 크다.**
- 앞 코드는 java8부턴 컴파일되나 java7까진 타입 추론 능력이 부족하여 반환 타입을 명시해야 한다.
    - 올바른 타입 추론을 못하면 명시적 타입 인수를 사용하여 타입을 알려주면 된다.
    
    ```java
    Set<Number> numbers = Union.<Number>union(integers,doubles);
    ```
        
- 펙스 공식을 두 번 적용한 예시
    - 기존 코드
    
    ```java
    public static <E extends Comparable<E>> E max(List<E> list)
    ```
    
    - 적용 후
    
    ```java
    public static <E extends Comparable<? super E>> E max(List<? extends E> list)
    ```
    
    - 위 메소드는 List를 받아 E 인스턴스를 생산하므로 입력 매개변수를 <? extends E>로 변경한다.
    - `Comparable<E>`는 E 인스턴스를 소비하므로 <? super E>로 대체한다.
        - Comparable은 언제나 소비자이므로 Comparable<? super E>를 적용하는 편이 낫다.
        - Comparator도 Comparator<? super E>를 적용하는 편이 낫다.
    - 클라이언트 코드 예시
    
    ```java
    List<ScheduledFuture<?>> scheduledFutures = ...;
    ```
    
    - 수정 전 max 메소드는 처리할 수 없다.
        - ScheduledFuture 는 `Comparable<ScheduledFuture>`를 구현하지 않았기 때문
        - `<E extends Comparable<E>>` 타입 매개변수 정의에 성립하지 않는다.
    - ScheduledFuture 는 Delayed의 하위 타입이다.
        - Delayed는 `Comparable<Delayed>` 를 구현하고 있다.
        - 즉, ScheduledFuture는 본인 타입 뿐 아니라 Delayed 인스턴스와도 비교할 수 있다.
    - Comparable(혹은 Comparator)을 직접 구현하지 않고, 직접 구현한 다른 타입(Delayed)을 확장한 타입(ScheduledFuture)을 지원하기 위해선 와일드카드를 적용해야 한다.

### 타입 매개변수와 와일드카드 둘 다 괜찮은 경우

> 기본 규칙: 메소드 선언에 타입 매개변수가 한 번만 나오면 와일드카드로 대체하라.
> 
- 비한정적 타입 매개변수, 비한정적 와일드카드 타입 적용한 swap 예시
    
    ```java
    // 주어진 리스트에서 명시한 두 인덱스의 아이템을 교환하는 swap 메소드
    public static <E> void swap(<List<E> list, int i, int j);
    public static void swap(List<?> list, int i, int j);
    ```
    
- 두 번째 비한정적 와일드카드 타입으로 선언한 swap 메소드의 컴파일 문제
    
    ```java
    public static void swap(List<?> list, int i, int j){
    	list.set(i, list.set(j, list.get(i));
    }
    ```
    
    - 위 코드는 컴파일 오류가 발생한다.
    - list.get(i) 에서 꺼내온 Object 인스턴스는 ? 형과 호환이 불가하다.
        - List<?>는 null 외에는 어떤 값도 넣을 수 없다.
        - 즉, read-only 로만 가능하다.
    - **해결: 와일드카드 타입의 실제 타입을 알려주는 메소드를 private 도우미 메소드로 작성하여 활용하라**
    
    ```java
    public static void swap(List<?> list, int i, int j){
    	swapHelper(list, i, j);
    }
    
    // 와일드카드 타입을 실제 타입으로 바꿔주는 private 도우미 메서드
    private static <E> void swapHelper(List<E> list, int i, int j){
    	list.set(i, list.set(j, list.get(i));
    }
    ```
    
    - 도우미 메소드는 제네릭 메소드여야 한다.
    - 리스트에서 꺼낸 타입은 항상 E 타입이며 E 타입을 넣어도 안전함이 보장된다.
    - 내부가 조금 복잡해지지만, 외부 API로는 와일드카드 기반의 선언을 유지할 수 있다.
        - 즉, 클라이언트는 복잡한 구현은 모른 채 와일드카드의 혜택(유연성!!)을 누릴 수 있다.
    

## 요약 정리

- 조금 복잡해도 와일드카드 타입을 적용하면 API가 훨씬 유연해진다.
- 따라서 널리 쓰일 라이브러리 코드를 작성한다면 와일드카드 타입을 적절히 사용해라
- PECS 공식을 기억해라
    - producer-extends
    - consumer-super
- Comparable과 Comparator는 모두 소비자다