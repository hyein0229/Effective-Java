# Item 29. 이왕이면 제네릭 타입으로 만들라
## 제네릭 클래스로 만드는 방법

### Object 기반 스택

- 이 클래스는 원래 제네릭 타입이어야 마땅하다.
- 지금 상태에선 클라이언트는 스택에서 꺼낸 객체를 형변환해주어야 하므로 런타임 오류의 가능성이 높다.

```java
public class Stack {
    private Object[] elements;
    private mnt size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[-size];
        elements[size] = null; //다쓴참조해제 
        return result;
    }

    public boolean isEmpty() {
        return size = 0;
    }

    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

### 첫 단계는 클래스 선언에 타입 매개변수를 추가하는 일이다.

- 스택이 담을 원소의 타입 하나만 추가하면 된다.
- Object 를 타입 매개변수 E로 바꾼다.
- 타입 이름으로는 보통 E를 사용한다.

```java
// 컴파일 되지 않는 코드 클래스에 <E> 추가 
public class Stack<E> {
    private E[] elements; // Object[] -> E[]
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        // Object -> E
        elements = new E[DEFAULT_INITIAL_CAPACITY]; // 컴파일 오류 발생
    }

    // 메소드 인수에서 Object -> E
    public void push(E e) {
        ensureCapacity();
        elements[size++] = e;
    }

    // Object -> E
    public E pop() {
        if (size = 0)
            throw new EmptyStackException();

        // Object -> E
        E result = elements[-size];
        elements[size] = null; //다쓴참조해제 
        return result;
    }
    // isEmpty와ensureCapacity메서드는그대로다. 
}
```

- new E[DEFAULT_INITIAL_CAPACITY]; 에서 컴파일 오류가 발생한다.
- **E와 같은 실체화 불가 타입으로는 배열 생성이 불가하다. → 문제 발생**

### 배열을 제네릭으로 만들 때의 문제 해결책은 두 가지가 있다.

- **첫 번째, 제네릭 배열 생성 금지 제약을 대놓고 우회하는 방법이 있다.**
    
    ```java
    // 1. 제네릭 배열로 형변환한 후 비검사 형변환 경고를 숨기는 방식
    public class Stack<E> {
        // ...
    
        // 배열 elements 는 push(E)로 넘어온 E 인스턴스만 담는다.
        // 따라서 타입 안전성을 보장하지만, 이 배열의 런타임 타입은 E[]가 아닌 Object[]이다.
        @SuppressWarnings("unchecked")
        public Stack() {
            elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
        }
        // ...
    }
    ```
    
    - 먼저, new E를 new Object 배열로 생성한다.
    - 그 다음 제네릭 배열(E[])로 형변환한다.
    - 컴파일 오류는 나지 않지만 unchecked cast 경고가 뜬다. (아이템 28 내용)
    - 비검사 형변환이 프로그램의 타입 안전성을 해치지 않음 스스로 확인해야 한다.
        - elements 배열은 private 필드이며 클라이언트에게 전달되는 일이 전혀 없다.
        - push 메소드로 배열에 저장되는 원소 타입은 항상 E다.
        - 따라서, 확실히 안전하다.
    - **타입 안전성을 검증했다면 경고를 숨기고 사용하라**
    - 범위를 최소로 좁혀 @SuppressWarnings 어노테이션으로 경고를 숨긴다.
    - 생성자가 배열 생성하는 일 말고는 다른 코드가 없으므로 생성자 전체에 적용해도 좋다.
    - 경고 없이 깔끔히 컴파일되며, 명시적 형변환 없이 런타임 형변환 오류가 발생하지 않는다.
- **두 번째, elements 필드의 타입을 E[]에서 Object[]로 바꾼다.**
    
    ```java
    public E pop() {
            if (size = 0)
                throw new EmptyStackException();
    
            // Object -> E incompatible types 컴파일 오류 발생
            E result = elements[-size];
            elements[size] = null; //다쓴참조해제 
            return result;
        }
    ```
    
    - pop 메소드에서 원소를 꺼내 E 타입에 할당하면 incompatible types 컴파일 오류가 발생한다.
    - E로 형변환해보자
    
    ```java
    // 2. Object[] 로 바꾸고 또 발생하는 비검사 형변환 경고를 숨겨라.
    public class Stack<E> {
        // 비검사 경고를 적절히 숨긴다.
        public E pop() {
            if (size == 0)
                throw new EmptyStackException();
    
            // push에서 E 타입만 허용하므로 이 형변환은 안전하다.
            @SuppressWarnings("unchecked")
            E result = (E) elements[--size];
    
            elements[size] = null;
            return result;
        }
    }
    ```
    
    - 해결을 위해 배열이 반환한 원소를 E로 형변환하면 unchecked cast 경고가 뜬다.
    - E는 실체화 불가 타입이므로 컴파일러는 런타임에 이뤄지는 형변환이 안전한지 보장하지 못한다.
    - pop 메소드 전체에 경고를 숨기지 말고, 비검사 형변환을 수행하는 할당문에만 적용시킨다.
        - 가능한 좁은 범위로 경고를 숨기기 위해

**두 가지 방식 비교**

- 첫 번째 방법은 가독성이 더 좋다.
    - 배열의 타입을 E[]로 선언하여 오직 E 타입 인스턴스만 받음을 어필할 수 있다.
    - 코드도 더 짧다.
    - 형변환도 배열 생성 시 딱 한번만 이루어진다.
- 두 번째 방식에선 배열에서 원소를 읽을 때마다 형변환을 해야 한다.
    - 따라서, 현업에선 첫 번째 방식을 더 선호하며 자주 사용한다.
    - 하지만, 배열의 런타임 타입이 컴파일타임 타입과 달라 힙 오염을 일으킨다.
    - **힙 오염** 이 마음에 걸리면 두 번째 방식을 사용하기도 한다.

___

**`힙 오염 (heap pollution)`**

JVM 힙 메모리 영역에 저장되어 있는 특정 변수(객체)가 불량 데이터를 참조함으로써, 힙에서 데이터를 가져오려 할 때 얘기치 못한 런타임 에러가 발생할 수 있는 상태를 말한다.

힙 오염의 대표 원인이 **제네릭**이다. 즉, 위 코드에선 컴파일타임에는 E타입이라고 했는데 실제로 런타임에는 E가 아닌 타입이 들어가 있는 상태를 말한다.

```java
ArrayList<String> list1 = new ArrayList<>();
list1.add("홍길동");
list1.add("임꺾정");

// 로직 수행...
Object obj = list1;
// 로직 수행...

// ArrayList<String> -> ArrayList<Double> 타입 캐스팅
// 실제 형변환 대상을 체크하지 않으므로 컴파일은 정상 동작
ArrayList<Double> list2 = (ArrayList<Double>) obj;
list2.add(1.0);
list2.add(2.0);

System.out.println(list2); // [홍길동, 임꺾정, 1.0, 2.0] 문자열과 double 이 공존

// 문자열 -> double cast 오류 발생
for(double n : list2) {
    System.out.println(n);
}
```

- 컴파일러는 타입 캐스팅을 검사하지 않는다는 것을 주의하라
    - 형변환 대상 객체에 대해 형변환을 할 수 있는지를 검사하지 않는다.
    - 컴파일러는 캐스팅하였을 때 대입되는 변수에 저장할 수 있는지만 확인한다.
    - 따라서, ArrayList<String> 를 ArrayList<Double> 타입으로 캐스팅이 가능하다.
- 제네릭은 타입 정보가 소거되어 어느 원소든 저장할 수 있게 된다.


## 제네릭 클래스 Stack의 사용

```java
public class Stack<T> {
    public static void main(String[] args) {
        Stack<String> stack = new Stack<>();
        for (String arg : args) {
            stack.push(arg);
        }
        while (!stack.isEmpty()) {
            System.out.println(stack.pop().toUpperCase()); // 명시적 형변환 없음
        }
    }
}
```

- stack.pop()에서 꺼낸 원소들을 명시적 형변환 없이 안전하게 사용한다.
- 이 형변환이 제네릭 사용함을 통해 항상 성공함을 보장할 수 있게 된다.
- 이는 “배열보단 리스트를 사용해라” 라는 아이템 28의 내용과는 모순되어 보인다.
    - 제네릭 타입 안에서 리스트를 사용하는 것이 항상 가능하지도, 항상 더 좋은 것은 아니다.
    - 자바가 리스트를 기본 타입으로 제공하지 않으므로 ArrayList 같은 제네릭 타입도 결국 기본 타입인 배열을 사용해 구현해야 한다.
    - 또한 HashMap 같은 제네릭 타입은 성능을 높일 목적으로 배열을 사용하기도 한다.
    - 즉, 제네릭 타입도 결국엔 내부 구현이 배열로 되어 있는 경우가 많다는 뜻인듯..?
- Stack 예처럼 대다수의 제네릭 타입은 타입 매개변수에 아무런 제약을 두지 않는다.
    - `Stack<int[]>`, `Stack<List<String>>` 등 어떤 참조 타입으로도 Stack을 만들 수 있다.
    - 단, 기본 타입은 사용할 수 없다.
    - Stack<int> 는 불가하다.
    - 기본 타입 박싱된 기본 타입을 이용해서 우회하여 사용할 수 있다.
- 타입 매개변수에 제약을 두는 제네릭 타입도 있다.
    - 예시로 java.util.concurrent.DelayQueue가 있다.
    
    ```java
    public class DelayQueue<E extends Delayed> implements BlockingQueue<E>
    ```
    
    - java.util.concurrent.Delayed 의 하위 타입만 받는 제네릭 타입이다.
    - 이렇게 하여 클라이언트는 형변환 없이 곧바로 Delayed 클래스의 메소드를 호출할 수 있다.
    - ClassCastException 오류를 걱정할 필요가 없다.
    - 이러한 타입 매개변수를 **한정적 타입 매개변수(bounded type paramter)** 라고 한다.

## 요약 정리

- 클라이언트에서 직접 형변환해야 하는 타입보다 제네릭 타입이 더 안전하고 쓰기 편하다.
- 따라서, 새로운 타입을 설계할 때는 형변환 없이도 사용할 수 있도록 하라.
- 그러기 위해선 제네릭 타입으로 만들어야 할 경우가 많다.
- 기존 타입 중 제네릭이어야 하는 것들은 제네릭 타입으로 변경해라.
- 기존 클라이언트 코드에 영향을 주지 않으면서, 새로운 사용자를 편하게 해주는 길이다.