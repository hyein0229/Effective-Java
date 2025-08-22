# Item 42. 익명 클래스보다는 람다를 사용하라
## 함수 타입의 표현 방식

- JDK 1.1 이전엔 자바에서 함수 타입을 표현할 때 추상 메서드를 하나만 담은 인터페이스를 사용했다.
    - 이런 인터페이스의 인스턴스를 함수 객체(function object)라고 한다.
- JDK 1.1 등장하며 함수 객체를 만드는 주요 수단은 **익명클래스**였다.
    
    ```java
    // 문자열을 길이순으로 정렬하기 위한 함수로 익명 클래스를 사용한 예시
    Collections.sort(words, new Comparator<String>() {
        public int compare(String s1, String s2) {
            return Integer.compare(s1.length(), s2.length());
        }
    });
    ```
    
    - 익명 클래스는 전략 패턴 구현에 주로 사용된다.
    - Comparator 인터페이스는 정렬을 담당하는 추상 전략이며, 구체 전략을 익명 클래스로 구현했다.
    - 하지만 단점은? 익명 클래스 방식은 코드가 너무 길기 때문에 함수형 프로그래밍에 적합하지 않다.
- Java 8에선 **추상 메서드 하나만 있는 인터페이스를 함수형 인터페이스**라고 부르게 되었다.
    - 함수형 인터페이스의 목적은 람다식으로 함수형 프로그래밍을 구현하기 위함이다.
    - 이는 람다 함수를 파라미터로 받을 때의 람다 함수의 타입명을 미리 지정해 제공한다.
    
    ```java
    @FunctionalInterface
    public interface Runnable {
        public abstract void run(); 
    }
    ```
    
    ```java
    // Thread 클래스 생성자에서 Runnable 타입을 받는다.
    Thread thread = new Thread( () -> {
        for(int i = 0; i < 10; i++) {
            System.out.println(i);
        }
    } );
    ```
    
    - @FunctionalInterface 어노테이션은 없어도 함수형 인터페이스로 동작은 한다.
    - @FunctionalInterface 의 역할은?
        - 람다식으로 사용할 의도임을 명시한다.
        - 추상 메소드가 2개 이상인지 인터페이스 검증하고, 아니면 컴파일 오류를 낸다.
    - 추상 메소드가 하나면 된다 → 즉, default 나 static 메소드는 여러 개 있어도 상관 없다.
- 함수형 인터페이스들의 인스턴스를 **람다식(lambda expression)** 을 사용해 만들 수 있게 된다.
    
    ```java
    Collections.sort(words,
            (s1, s2) -> Integer.compare(s1.length(), s2.length()));
    ```
    
    - 람다에 비교자 생성 메서드를 사용하면 더 간결하게 만들 수 있다.
        - 비교자 생성 메서드란? 기준값을 추출하여 Comparator를 만들어 반환하는 도우미 메서드
    
    ```java
    // Comparator.comparingInt
    Collections.sort(words, comparingInt(String::length));
    ```
    
    - Java 8 때 List 인터페이스에 추가된 sort 메서드 사용
    
    ```java
    words.sort(comparingInt(String::length));
    ```
    

## 람다(lambda)

- 람다는 함수형 인터페이스의 구현체로 익명 함수이다.
- 함수나 익명 클래스와 개념은 비슷하나 코드는 훨씬 간결해진다.
- 코드가 깔끔해지고 어떤 동작을 하는지가 명확히 드러난다.

### 람다의 타입 추론

- 람다, 매개변수, 반환값의 타입은 명시하지 않아도 컴파일러가 문맥을 살펴 타입 추론을 해준다.
- 만약, 컴파일러가 타입을 결정하지 못한다면 프로그래머가 직접 명시해주어야 한다.
- 타입 추론 규칙은 많고 복잡하다.
    - 따라서, **타입을 명시해야 코드가 더 명확한 경우만 제외하면 람다의 모든 매개변수 타입은 생략하라.**
    - 컴파일러가 타입 추론을 못하는 오류가 나타날 때만 타입을 명시하면 된다.


**제네릭과 람다**

- item 26: 제네릭의 로 타입을 쓰지 말라
- item 29: 제네릭을 써라
- item 30: 제네릭 메서드를 써라

위 조언은 람다와 함께 쓸 때 중요하다. 

컴파일러가 타입을 추론하는 데 필요한 타입 정보 대부분을 제네릭에서 얻기 때문이다. 

예를 들어 인수 words 가 `List<String>`이 아니라 로 타입 `List` 였다면 컴파일 오류가 났을 것 이다.

___

### 열거타입에서의 람다 예제

- 상수별 클래스 몸체를 사용한 각기 달리 동작하는 코드 구현 예시
    
    ```java
    public enum Operation {
        PLUS("+") {public double apply(double x, double y) {return x + y;}},
        MINUS("-") {public double apply(double x, double y) {return x - y;}},
        TIMES("*") {public double apply(double x, double y) {return x * y;}},
        DIVIDE("/") {public double apply(double x, double y) {return x / y;}};
    
    	private final String symbol;
    
        Operation(String symbol) { this.symbol = symbol; }
    
        @Override public String toString() { return symbol; }
    
        // 추상 메서드
        public abstract double apply(double x, double y);
    }
    ```
    
    - 상수별 클래스 몸체 구현 방식보단 열거 타입에 인스턴스 필드를 두는 편이 낫다. (item 34)
- 람다를 사용하여 인스턴스 필드 방식으로 구현한 예시
    - 람다를 사용하면 인스턴스 필드를 이용할 수 있다.
    
    ```java
    public enum Operation {
        PLUS  ("+", (x, y) -> x + y),
        MINUS ("-", (x, y) -> x - y),
        TIMES ("*", (x, y) -> x * y),
        DIVIDE("/", (x, y) -> x / y);
    
        private final String symbol;
        private final DoubleBinaryOperator op; // 인스턴스 필드 추가
    
        Operation(String symbol, DoubleBinaryOperator op) {
            this.symbol = symbol;
            this.op = op;
        }
    
        @Override public String toString() { return symbol; }
    
        public double apply(double x, double y) {
            return op.applyAsDouble(x, y);
        }
    }
    ```
    
    - 단순히 열거 타입 상수의 동작을 람다로 구현해 생성자에 넘기면 된다.
    - 열거 타입에 인스턴스 필드를 두고 생성자를 통해 받은 람다식을 필드에 저장한다.
    - 다음 apply 메서드에선 필드에 저장된 람다를 호출하면 된다.
    - DoubleBinaryOperator 인터페이스?
        - java.util.function 에서 제공하는 다양한 함수형 인터페이스 중 하나이다.
        - double 타입 인수 2개를 받아 double 타입을 반환한다.

___

### 람다의 제약

- **메서드나 클래스와 달리, 람다는 이름이 없고 문서화도 못한다.**
- 따라서, **코드 자체로 동작이 명확히 설명되지 않거나 코드 줄 수가 많아지면 람다를 쓰면 안된다.**
    - 람다는 한 줄이 가장 좋고 최대 3줄 안에 끝나는 것이 좋다.
    - 그 이상은 가독성이 심하게 나빠져 짧게 리팩토링하거나 쓰지 말아라
- **열거 타입 생성자 안의 람다는 열거 타입의 인스턴스 멤버에 접근할 수 없다.**
    - 열거 타입 생성자에 넘겨지는 인수는 컴파일타임에 추론된다.
    - 하지만, 인스턴스는 런타임에 생성되므로 접근이 불가하다.
- **람다는 함수형 인터페이스에만 쓰인다.**
    - 즉, 함수형 인터페이스가 아닌 곳에선 람다를 사용할 수가 없다.
    - 따라서 추상 클래스의 인스턴스를 만들 때는 익명 클래스를 사용해야 한다.
        
        ```java
        abstract class Animal { // 추상 클래스
            abstract void sound();
            void sleep() { System.out.println("sleeping"); }
        }
        
        Animal cat = new Animal() {
            @Override
            void sound() {
                System.out.println("야옹");
            }
        };
        ```
        
    - 추상 메서드가 여러 개인 일반 인터페이스의 인스턴스를 만들 때도 익명 클래스를 써야 한다.
        
        ```java
        // 추상 메서드가 2개인 인터페이스
        interface MultiMethodInterface {
            void a();
            void b();
        }
        
        MultiMethodInterface m = new MultiMethodInterface() {
            @Override
            public void a() { System.out.println("a"); }
            @Override
            public void b() { System.out.println("b"); }
        };
        ```
        
- **람다는 자신을 참조할 수 없다.**
    - 람다에서의 this 키워드는 바깥 인스턴스를 가리킨다.
    - 익명 클래스에서의 this는 익명 클래스의 인스턴스 자신을 가리킨다.
    - 따라서, 함수 객체가 자신을 참조해야 한다면 반드시 익명 클래스를 사용해야 한다.
- **람다도 익명 클래스처럼 직렬화 형태가 구현별로(가령 JVM별로) 다를 수 있다.**
    - 따라서 람다를 직렬화하는 일은 극히 삼가야 한다.
    - 직렬화해야 하는 함수 객체가 있다면(Comparator처럼) private 정적 중첩 클래스 인스턴스를 사용해라

## 요약 정리

- Java 8부터 작은 함수 객체를 구현하는 데 적합한 람다가 도입되었다.
- 익명 클래스는 함수형 인터페이스가 아닌 타입의 인스턴스를 만들 때만 사용하여라
- 람다는 함수 객체를 아주 간결하게 표현할 수 있어 자바에서의 함수형 프로그래밍의 지평을 열었다.