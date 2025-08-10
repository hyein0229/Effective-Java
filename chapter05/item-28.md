# Item 28. 배열보다는 리스트를 사용하라
## 배열 vs 제네릭 타입

- **첫째, 배열은 공변이고, 제네릭은 불공변이다.**
    - A가 B의 하위 타입이라면 배열 A[]는 배열 B[]의 하위 타입이 된다. (즉, 함께 변한다는 뜻이다.)
        
        ```java
        Object[] objectArray = new Long[1]; // 런타임에 타입이 실체화되어 컴파일은 성공
        objectArray[0] = "타입"; // 그러나 런타임에 ArrayStoreException 발생
        ```
        
    - `List<Type1>`은 `List<Type2>`의 하위 타입도, 상위 타입도 아니다.
        
        ```java
        // 컴파일 불가, 호환되지 않는 타입
        List<Object> ol = new ArrayList<Long>(); 
        ol.add("타입");
        ```
        
    - 즉, 위 두 코드에서 배열의 경우는 런타임에야 Long 배열에 String을 넣는 오류를 알 수 있다.

- **둘째, 배열은 실체화되는 반면, 제네릭은 타입 정보가 런타임에는 소거된다.**
    - 배열은 런타임에도 자신이 담기로 한 원소의 타입을 인지하고 확인한다.
    - 따라서 배열은 Long 배열에 String을 넣으려고 할 때 ArrayStoreException이 발생하였다.
    - 제네릭은 원소 타입을 컴파일시에만 검사하면 런타임에는 알수조차 없다.
    - 제네릭의 타입 소거는 제네릭 이전 레거시 코드와 함께 사용할 수 있도록 호환을 위한 메커니즘이다.

## 이러한 차이로 배열과 제네릭은 잘 어우러지지 못한다.

- 배열은 런타임에도 타입을 알아야 하고, 제네릭은 런타임에 타입을 모른다. → 같이 사용이 어렵다.
- 배열은 제네릭 타입, 매개변수화 타입, 타입 매개변수로 사용할 수 없다.
    - 즉, `new List<E>[]`, `new List<String>[]`, `new E[]` 식으로 작성이 불가하다.
- 제네릭 배열을 만들 수 없도록 한 이유는?
    - 타입 안전하지 않기 때문이다.
    - 제네릭 배열을 허용하면 컴파일러의 형변환 코드에서 런타임에 ClassCastException 이 발생할 수 있다.
    - 런타임에 ClassCastException 발생을 막아주겠다는 제네릭 타입 시스템의 취지에 어긋난다.

제네릭 배열이 허용된다고 가정하고 예제를 보자.

```java
class Example {
    public static void main(String[] args) {
        List<String>[] stringLists = new List<String>[1]; // 허용된다고 가정
        List<Integer> intList = List.of(42);
        Object[] objects = stringLists;
        objects[0] = intList;
        String s = stringLists[0].get(0); // 런타임 ClassCastException 발생
    }
}
```

- List<Integer> intList = List.of(42);
    - 원소가 하나인 리스트를 생성한다.
- Object[] objects = stringLists;
    - 배열은 공변이다.
    - 즉, List는 Object의 하위 타입이므로 List 배열을 Object 배열에 할당한다.
- objects[0] = intList;
    - (2)에서 생성한 List<Integer> 인스턴스를 첫 원소로 저장한다.
    - 제네릭은 타입 소거되므로 List[] 에 List 인스턴스를 담는 것과 같다.
    - 따라서 ArrayStoreException을 일으키지 않는다.
- String s = stringLists[0].get(0);
    - 실제로 배열 첫 원소엔 List<Integer> 인스턴스가 담겨 있다.
    - **컴파일러가 꺼낸 원소 Integer를 자동 형변환하려고 할 때 ClassCastException가 발생한다.**
    - 따라서, 예외 방지를 위해선 제네릭 배열이 생성되지 않도록 컴파일 오류 내야 한다.
    

## 실제화 불가 타입(non-reifiable type)

- E, List<E>, List<String> 같은 타입
- **실체화되지 않아 런타임에는 컴파일타임보다 타입 정보를 적게 가지는 타입**을 뜻한다.
- 소거 매커니즘 때문에 매개변수화 타입 가운데 실체화될 수 있는 타입은 List<?>와 Map<?,?> 같은 비한정적 와일드카드 타입뿐이다.
    
```java
// 컴파일 에러가 발생 안함
List<?>[] lists = new List<?>[2];
lists[0] = Arrays.asList(1); // 모든 제네릭 타입을 가질 수 있음
lists[1] = Arrays.asList("A");
    for (List<?> list : lists) {
        System.out.println(list);
    }
}
```
    
- 배열을 비한정적 와일드카드 타입으로 만들 순 있지만, 유용하게 쓰일 일이 거의 없다.
- 배열을 제네릭으로 만들 수 없어 귀찮을 때도 있다.
    - 제네릭 컬렉션에서는 자신의 원소 타입을 담은 배열을 반환하는게 보통은 불가능하다.
        
        ```java
        List<String> list = new ArrayList<>();
        String[] arr = list.toArray(); // 배열로 바꾸기 시도, 컴파일 오류
        ```
        
    - 제네릭 타입과 가변인수 메서드(varargs method)를 함께 쓰면 해석하기 어려운 경고 메시지를 받게 된다.
    - 가변인수 메서드를 호출할 때마다 가변인수 매개변수를 담을 배열이 하나 만들어지는데, 이때 그 배열의 원소가 실체화 불가 타입이라면 경고가 발생하는 것이다.
        - 이 문제는 @SafeVarargs 어노테이션으로 대처할 수 있다.
        
        ```java
        @SafeVarargs
        static <T> void printAll(T... elements) { // 가변인수 메소드
            for (T e : elements)
                System.out.println(e);
        }
        
        printAll(new ArrayList<String>(), new ArrayList<String>()); // 제네릭 사용
        ```
        

## 형변환 오류나 경고가 뜬다면 배열보단 리스트를 사용하라

- 배열로 형변환하려할 때
- 제네릭 배열 생성 오류나 비검사 형변환 경고가 뜨는 경우 배열인 E[] 대신 컬렉션인 `List<E>`를 사용하면 해결된다.
- 코드가 조금 복잡해지고 성능이 살짝 나빠질 수도 있지만, 타입 안전성과 상호운용성은 좋아진다.
    
```java
// 제네릭을 쓰지 않은 예제
public class Chooser {
    private final Object[] choiceArray;
        
    // 생성자에서 컬렉션을 받아 배열로 변환
    public Chooser(Collection choices) {
        choiceArray = choices.toArray(); // ?
    }
        
    // 해당 메서드 호출 시 반환된 Object를 원하는 타입으로 형변환 필요
    public Object choose() {
        Random rnd = ThreadLocalRandom.current();
        return choiceArray[rnd.nextInt(choiceArray.length)];
    }
}
```
    
- choose 메소드에서 Object 반환 시 항상 원하는 타입으로 형변환해서 사용한다.
- 이때, 다른 원소가 들어 있다면 런타임에 형변환 오류가 발생한다.
- 제네릭으로 만들어 자동 형변환할 수 있도록 하자
    
```java
// 제네릭으로 리팩토링한 예제
public class Chooser<T> {
    private final T[] choiceArray;

    public Chooser(Collection<T> choices) {
        choiceArray = choices.toArray(); // 컴파일 시 형변환 오류가 발생한다.
    }
    
    // .. choose 메소드는 같다.
}
```
    
- incompatible types: `Object[] cannot be converted to T[]`
- 애초에 형변환 자체가 없으므로 컴파일하지 못한다.
- Object 배열을 T 배열로 형변환해보자.
    

```java
public class Chooser<T> {
    private final T[] choiceArray;
        
    // T[] 타입으로 형변환
    public Chooser(Collection<T> choices) {
        choiceArray = (T[]) choices.toArray(); // unchecked cast 경고가 발생한다.
    }
}
```
    
- T가 무슨 타입인지 모르므로 컴파일러는 이 형변환이 런타임에도 안전한지 보장할 수 없다는 경고 메시지를 띄운다.
- 제네릭에선 원소의 타입 정보가 소거되어 런타임에 무슨 타입인지 전혀 알 수 없다는 것을 기억하자
- 위 코드는 동작은 하지만 컴파일러가 안전은 보장하지 못한다.
- 코드가 안전하다고 확신하면 주석을 남기고 경고를 숨기는 어노테이션을 달아도 된다.
- 하지만, 애초에 경고의 원인을 제거하는 편이 훨씬 낫다.
- **비검사 형변환 경고를 제거하려면 배열 대신 리스트를 쓰면 된다.**
    
```java
public class Chooser<T> {
    private final List<T> choiceList; // 배열이 아닌 리스트 사용

    public Chooser(Collection<T> choices) {
        choiceList = new ArrayList<>(choices);
    }

    public T choose() {
        Random rnd = ThreadLocalRandom.current();
        return choiceList.get(rnd.nextInt(choiceList.size));
    }
}
```
    
- 오류나 경고 없이 컴파일되며 정상 동작한다.
- 코드 양도 조금 늘고 조금 느리지만, 런타임에 ClassCastException 을 만날 일이 없으니 안전하다.

## 요약 정리

- 배열과 제네릭은 매우 다른 타입 규칙이 적용된다.
- 배열은 공변이고 실체화되는 반면, 제네릭은 불공변이고 타입 정보가 소거된다.
- 배열은 런타임에는 타입 안전하지만 컴파일타임에는 그렇지 않다.
- 제네릭은 컴파일타임에는 안전하지만 런타임에는 그렇지 않다.
- 둘의 차이로 인해 둘을 같이 쓰는 것은 어렵다.
- 둘을 섞어 쓰다가 컴파일 오류나 경고를 만나면, 가장 먼저 배열을 리스트로 대체하는 방법을 적용해라