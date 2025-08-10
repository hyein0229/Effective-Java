# Item 32. 제네릭과 가변인수를 함께 쓸 때는 신중하라

## 가변인수(varargs) 메소드

- 자바 5 때 제네릭과 함께 추가되었다.
- 가변인수는 메소드에 넘기는 인수의 개수를 클라이언트가 조절하는 유연성을 제공하는 것이 장점이다.
- 구현 방식은 가변인수 메소드는 호출 시 가변인수를 담기 위한 배열이 자동으로 하나 만들어진다.
- 구현 방식의 문제점
    - varargs 매개변수에 제네릭이나 매개변수화 타입이 포함되면 컴파일 경고가 발생한다.
    - item28에서 실체화 불가 타입은 런타임엔 타입 정보가 소거되는 것을 알았다.
        - **배열은 공변, 제네릭은 불공변하다.**
    - 따라서 메소드 선언 시 실체화 불가 타입으로 varargs 매개변수 사용 시 **힙 오염** 경고를 보낸다.
- **힙 오염(heap pollution)이란?**
    - 매개변수화된 타입의 변수가 다른 타입의 객체를 참조할 때 발생한다.
    - 다른 타입 객체를 참조하는 상황에선 컴파일러가 자동 생성한 형변환이 실패할 수 있다.
    - 즉, 제네릭 타입 시스템이 약속한 타입 안전성이 흔들린다.
- 제네릭과 varargs를 혼용하면 타입 안전성이 깨진다.
    
```java
static void dangerous(List<String>... stringLists){ // 가변인수 메소드
    List<Integer> intList = List.of(42); // (1)
    Object[] objects = stringLists; // (2)
    objects[0] = intList; // (3) 힙 오염 발생
    String s = stringLists[0].get(0); // (4) ClassCastException
}
```
    
- 가변인수 메소드는 List<String>[] 배열 stringLists을 만들어낸다.
- Object[] 형인 objects 에 stringLists 를 대입한다.
    - 배열은 공변이기 때문에 가능하다.
    - List 는 Object 의 하위 타입이므로 배열도 하위 타입으로 동작한다.
- 따라서, 현재 objects 는 `List<String> `배열이 된다.
- objects[0]에 insList를 할당하면 `List<String>` 배열이 `List<Integer>` 원소를 가지게 된다.
    - 즉, **힙 오염이 발생한다!!!!!**
- 힙 오염으로 저장된 `List<Integer>` 원소를 꺼내어 String 자동 형변환 시 ClassCastException 발생
- 제네릭 배열을 직접 생성하지 못하게 하면서 제네릭 가변인수 메소드는 허용한 이유는 무엇일까?
    - 실무에서 매우 유용하므로 모순을 수용하기로 하였다.
    - 자바 라이브러리에서도 Arrays.asList(T… a), Collections.addAll 메소드 등이 있다.

## @SafeVarargs

- 자바 7 에서 추가되어 제네릭 가변인수 메소드 작성자가 클라이언트 측에서 발생하는 경고를 숨길 수 있다.
    - 7 이전엔 작성자가 호출자 쪽에서 발생하는 경고에 대해 할 수 있는 일이 없었다.
    - 사용자는 그냥 두거나 호출하는 곳마다 @SuppressWarnings 어노테이션을 달아야 했다.
    - 이는 지루하고, 가독성을 떨어트리고 진짜 문제를 숨길 가능성이 존재한다.
- @SafeVarargs 어노테이션은 **메소드 작성자가 그 메소드가 타입 안전함을 보장하는 장치다.**
- 메소드가 안전한지 어떻게 확신할까?
    - 일단, 메소드 호출 시 제네릭 배열이 만들어진다는 사실을 기억해라
    - 메소드에서 배열에 아무것도 저장하지 않고, 그 배열의 참조가 밖으로 노출되지 않는다면 타입 안전하다.
    - 즉, **제네릭 배열이 순수하게 호출자로부터 메소드로 인수를 전달하는 역할만 하는 것이다.**

**[자신의 제네릭 매개변수 배열의 참조를 노출하는 예시 코드]**

```java
static <T> T[] toArray(T... args){
	return args;
}
```

- 반환된 배열은 외부 어디서든 수정될 수 있다.
- 힙 오염을 메소드를 호출한 쪽의 콜스택으로 전이되는 결과를 낳는다.

```java
// 안전하지 않다.
static <T> T[] pickTwo(T a, T b, T c){
	switch (ThreadLocalRandom.current().nextInt(3)){
		case 0: return toArray(a, b);
		case 1: return toArray(a, c);
		case 2: return toArray(b, c);
	}
	throw new AssertionError(); // 도달할 수 없다.
}
```

- 컴파일러는 toArray에 넘어온 T 인스턴스들을 담을 Object[] 배열을 만드는 코드를 생성한다.
- pickTwo 에선 toArray 가 돌려준 Object[] 배열을 그대로 반환한다.

```java
public static void main(String[] args) {
	String[] attributes = pickTwo("빠른", "좋은", "저렴한"); // ClassCastException
}
```

- pickTwo의 반환값을 String[] 형에 저장하면서 컴파일러는 자동 형변환 코드를 생성한다.
- Object[]는 String[]의 하위 타입이 아니므로 형변환은 실패한다.
- 힙 오염을 발생시킨 진짜 원인은 toArray 로부터 두 단계나 떨어져있다.
- 즉, **제네릭 varargs 매개변수 배열에 다른 메소드가 접근하도록 허용하면 안전하지 않다는 것을 알 수 있다.**

### **제네릭 varargs 매개변수 배열에 다른 메소드가 접근해도 안전한 경우**

- 첫번째, @SafeVarargs로 제대로 애노테이트된 또 다른 가변인수 메소드에 넘기는 경우
    
    ```java
    // 임의 개수의 리스트를 받아 하나의 리스트로 옮겨 담아 반환한다.
    @SafeVarargs
    static <T> List<T> flatten(List<? extends T>... lists){
    	List<T> result = new ArrayList<>();
    	for(List<? extends T> list : lists){
    		result.addAll(list);
    	}
    	return result;
    }
    ```
    
- 두번째, 그저 배열 내용의 일부 함수를 호출만 하는 일반 메소드에 넘기는 경우

### @SafeVarargs 사용 규칙

- **제네릭이나 매개변수화 타입의 varargs 매개변수를 받는 모든 메소드에 @SafeVarargs를 달라**
- 즉, 안전하지 않은 가변인수 메소드는 절대 작성하지 말아라
- 다음 두 조건을 모두 만족하는 제네릭 varargs 메소드는 안전하다.
    - varargs 매개변수 배열에 아무것도 저장하지 않는다.
    - 그 배열(혹은 복제본)을 신뢰할 수 없는 코드에 노출하지 않는다.
- @SafeVarargs 어노테이션은 재정의 불가한 메소드에만 달아야 한다.
    - 재정의한 메소드도 안전한지는 보장할 수 없기 때문이다.
    - 자바 8에선 오직 정적 메소드와 final 인스턴스 메소드에만 붙일 수 있다.
    - 자바 9부턴 private 인스턴스 메소드에도 허용된다.

### @SafeVarargs이 유일한 정답은 아니다 - varargs 매개변수를 List 로 바꾼다

- 제네릭 varargs 매개변수를 List로 대체한 예시 코드
    
    ```java
    static <T> List<T> flatten(List<List<? extends T>>... lists){
    	List<T> result = new ArrayList<>();
    	for(List<? extends T> list : lists){
    		result.addAll(list);
    	}
    	return result;
    }
    ```
    
    ```java
    // List.of 로 리스트 인스턴스 생성하여 넘긴다.
    List<List<String>> audience = flatten(List.of(friends, romans, countrymen));
    ```
    
    - List.of 에도 임의 개수의 인수를 넘길 수 있다.
        - List.of() 도 @SafeVarargs 어노테이션이 달아있는 가변인수 메소드이다.
- 장점
    - 컴파일러가 이 메소드의 타입 안전성을 검증할 수 있다.
    - @SafeVarargs 어노테이션을 달지 않아도 된다.
    - 실수로 안전하다고 판단할 걱정이 없다.
- 단점
    - 클라이언트 코드가 살짝 지저분해진다.
    - 속도가 조금 느려질 수 있다.
- varargs 메소드를 안전하게 작성할 수 없는 경우에도 사용할 수 있다.
    
    ```java
    static <T> List<T> pickTwo(T a, T b, T c){
    	switch (ThreadLocalRandom.current().nextInt(3)){
    		case 0: return List.of(a, b);
    		case 1: return List.of(a, c);
    		case 2: return List.of(b, c);
    	}
    	throw new AssertionError();
    }
    ```
    
    ```java
    List<String> attributes = pickTwo("good", "fast", "cheap");
    ```
    
    - toArray의 List 버전인 List.of 를 사용한다.
    - 배열 없이 제네릭만 사용하므로 타입 안전성이 보장된다.

## 요약 정리

- 가변인수 메소드는 뛰어난 유연성을 제공한다.
- 그러나, 가변인수와 제네릭은 궁합이 좋지 않다.
    - 가변인수 기능은 배열을 노출하여 추상화가 완벽하지 못하다.
    - 배열은 공변하고 제네릭은 불공변하기 때문이다.
- 해결 방법 두 가지
    - 먼저 안전한지 확인 후 @SafeVarargs 달아라
    - 제네릭 varargs 매개변수를 List 로 바꿔라