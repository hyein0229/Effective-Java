# Item 08. finalizer 와 cleaner 사용을 피하라

어렵다 이 파트..

**자바에서 객체 소멸은 GC 가 담당하고,** 

**비메모리 자원 회수는 try-with-resources, try-finally 로 해결한다.**

여기서 비메모리 자원이란 파일, 소켓, db 커넥션, 네이티브 자원 같은 OS 자원을 말한다.

## Finalizer 란? 쓰지 말아야 하는 이유

finalize() 메서드는 java.lang.Object 클래스에 정의되어 있으며, 자바에서 객체가 GC 에 의해 제거될 때 실행된다. Finalizer 는 이 finalize 를 오버라이드한 것으로 객체가 소멸될 때 마지막으로 수행할 수 있는 작업을 정의하는 데 사용된다.

하지만, finalizer 는 예측할 수 없으며, 상황에 따라 위험할 수 있어 불필요하며, 오작동, 낮은 성능, 이식성 문제의 원인으로 기본적으로는 쓰지 말아야 한다. 그 대안인 cleaner 도 덜 위험하지만 여전히 예측 불가하며, 느리고, 불필요하다.

### 1. 즉시 수행된다는 보장이 없다.

객체에 접근할 수 없게 된 후, 즉 참조가 해제된 후 finalizer 나 cleaner 가 실행되기까지 얼마나 걸릴지 알 수 없다. 즉, 제때 실행되어야 하는 작업은 절대 할 수 없다.

얼마나 신속히 수행될지는 가비지 컬렉터 알고리즘에 전적으로 달려있다.

___

### 2. 자원 회수가 우선순위에 의해 지연된다.

finalizer 스레드는 다른 애플리케이션 스레드보다 우선순위가 낮아 실행될 기회를 얻지 못할 수 있다. 이로 인해 수천 개의 객체가 finalizer 대기열에서 회수되기만을 기다리다가 OOM 문제가 발생했던 실제 상황도 있다. 

cleaner 는 자신을 수행할 스레드를 제어할 수 있다는 면에서 나을 수 있으나 여전히 가비지 컬렉터 통제하에 있으며 즉각 수행이 보장되지 않는다.

___

### 3. 수행 시점뿐 아니라 수행 여부조차 보장되지 않는다.


접근할 수 없는 객체에 딸린 종료 작업이 전혀 수행되지 못한 채 프로그램이 중단될 수 도 있다. 

`System.gc`나 `System.runFinalization` 메서드는 실행 가능성을 높여줄 순 있으나 gc 와 finalize 실행 요청을 보낼 뿐 수행을 보장하진 않는다. `System.runFinalizersOnExit`와 `Runtime.runFinalizersOnExit`는 JVM 종료 전 강제 실행을 보장해주긴 하나 다른 스레드가 소멸 대상의 객체에 접근하여 사용 중이어도 실행해 버린다는 매우 심각한 결함이 있다.

___

### 4. finalizer 동작 중 발생한 예외는 무시된다.

finalizer 내부 로직에서 예외가 발생되어도 무시된다. 즉, 예외가 발생한 시점 이후의 코드는 실행되지 않아 자칫 마무리 작업이 덜 된 상태로 남을 수 있다. 스레드가 중단되지도, 로그가 뜨지도 않아서 개발자는 모른다. 이렇게 되면 다른 스레드에서 이렇게 훼손된 객체를 사용하게 될 수 있어 잘못된 동작을 할 수 있다. 

cleaner 는 자신의 스레드 통제가 가능하여 이런 문제는 발생하지 않는다고 한다.

___

### 5. 심각한 성능 문제를 동반한다.


finalizer 와 cleaner 는 가비지 컬렉터의 효율을 떨어트려 try-with-resources 에 비해 수거하기까지 굉장히 오랜 시간이 걸린다.

---

### 6. 보안 문제를 일으킬 수 있다.


finalizer를 사용한 클래스는 finalizer 공격에 노출되어 보안 문제가 발생할 수 있다. 

생성자나 직렬화 과정에서 예외가 발생하면, 이 생성되다 만 객체에서 악의적인 하위 클래스의 finalizer 가 수행될 수 있게 된다. 이 finalizer 는 정적 필드 (static) 에 자신의 참조를 할당하여 GC가 수거하지 못하도록 할 수 있다.

즉, 이러한 공격을 막으려면 상속 자체를 못하도록 final 클래스로 만들거나 아무 일도 하지 않는 finalize 메소드를 만들고 final 로 선언하여라.

**❓ 생성되다 만 객체가 어떻게 finalize 호출이 되는지 ?**

- 생성자 내부에서 예외 발생 → 생성이 완전히 안된 것 같지만?
- 실제로는 **JVM 내부적으로 해당 객체가 이미 힙에 partially allocated 상태로 존재함**
- 따라서 해당 객체는 GC 대상이며 finalize 가 호출될 수 있음
- 해당 finalize 내부에서 악의적으로 static 변수에 미완성된 자기 자신 (this) 를 할당할 수 있음

## Finalizer 의 대안 AutoCloseable

AutoCloaseable 을 구현해주고 클라이언트에서 인스턴스를 다 쓰고 난 뒤 close 메소드를 호출한다. 일반적으로 예외가 발생해도 제대로 종료될 수 있도록 try-with-resources 를 사용해야 한다. 

구현할 때 알아두면 좋은 점은 각 인스턴스는 자신이 닫혔는지를 추적할 수 있도록 하는 것이 좋다. 따라서 close 메소드 내부에서 자신이 더 이상 유효하지 않음을 필드에 기록하고, 다른 메소드에선 이 필드를 검사해서 close 된 이후라면 `IllegalStateException` 을 던지도록 하는 것이다.

### AutoCloseable 구현

```java
public class MyResource implements AutoCloseable {
    private boolean closed = false; // 상태 추적 

    public void doWork() {
        if (closed) {
            throw new IllegalStateException("리소스는 이미 닫혔습니다!");
        }
        System.out.println("작업 수행 중...");
    }

    @Override
    public void close() {
        closed = true;
        System.out.println("리소스 닫힘");
    }
}
```

### try-finally

```java
public class TryFinallyExample {
    public static void main(String[] args) {
        MyResource resource = new MyResource();
        try {
            resource.doWork();
        } finally {
            resource.close();
        }
    }
}
```

- finally 내부에서 예외 여부에 상관없이 close 호출을 보장
- 리소스를 쓰는 클라이언트 입장에서 명시적 close 메소드 호출이 필요

### try-with-resources (가장 좋은 방법)

```java
public class TryWithResourcesExample {
    public static void main(String[] args) {
        try (MyResource resource = new MyResource()) {
            resource.doWork();
        }  // 자동 close 호출
    }
}
```

- ***Autocloseable 을 구현한 객체의 경우 try 블록이 끝나는 시점에 자동 close 호출됨***

## 그러면 finalizer 나 cleaner 는 언제 쓰나

- 클라이언트가 close 호출하지 않았을 경우에 대비한 “안전망”의 역할
    - finalize 가 호출되리란 보장은 없겠지만 아무것도 안하는 것보단 낫다.
    - FileInputStream, FileOutputStream, ThreadPoolExecutor는 안전망 역할의 finalizer를 제공하는 라이브러리 중 하나다.
- *네이티브 피어*와 연결된 객체에서 사용
    - 네이티브 피어는 일반 자바 객체가 네이티브 메소드를 통해 기능을 위임한 객체이다. 
        - 즉, 자바 객체 내에서 자바 외 리소스 사용
    - 즉, 네이티브는 GC가 알지 못하니 자바 객체 회수 시 네이티브 객체까지 회수가 불가능하다.
    - 따라서, finalizer 나 cleaner 를 사용하기 적당한 작업이다. (이것도 안전망 역할인듯)
    - 그러나 성능 저하를 감당할 수 없거나 네이티브 피어가 사용하는 자원을 즉시 회수해야 하는 경우엔 close 메소드를 사용해야 한다.

 cleaner 를 안전망으로 활용하는 AutoCloseable 클래스

```java
import java.lang.ref.Cleaner;

public class Room implements AutoCloseable{
    private static final Cleaner cleaner = Cleaner.create();
	   
	  // Cleanable은 별도의 쓰레드로 clean을 함.
	  // Room을 참조하면 순환참조가 되어 GC 가 수거할 수 없음 
	  // 정적클래스가 아니면 자동으로 바깥객체의 참조를 가지므로 GC 가 수거할 수 없음
    private static class State implements Runnable {
        int numJunkPiles; // 방을 청소할 때 수거할 자원
        
        State(int numJunkPiles){
            this.numJunkPiles = numJunkPiles;
        }

        @Override
        public void run() {  // 1. close를 호출할 때, 2. cleaner(안전망)
            System.out.println("방 청소");
            numJunkPiles = 0;
        }
    }
    
    
    private final State state;
    private final Cleaner.Cleanable cleanable;
    
    
    public Room(int numJunkPiles) {
        state = new State(numJunkPiles);
        cleanable = cleaner.register(this, state);
    }
    
    @Override
    public void close(){
        cleanable.clean();
    }
            
}
```

**run 메소드가 호출되는 두 상황**

- Room 의 close 메소드 호출될 때
    - close 메소드에서 Cleanable 의 clean 호출 → clean 안에서 run 호출
    - try-with-resources 와 같이 사용하는 경우 자동 close 호출
- close 호출이 없는 경우 cleaner 가 (바라건대) run 호출
    - 반드시 호출되리란 보장은 없다. 없는 것보단 낫다는 것이다.

**try-with-resources 사용으로 자동 close 호출**

```java
public class Adult {
    public static void main(String[] args) {
        try (Room room = new Room(7)){
            System.out.println("Hi~"); // Hi~ 다음 close 호출되어 "방 청소"가 출력됨
        }
    }
}
```

 
**close 를 하지 못했을 경우**

```java
public class Teenager{
    public static void main(String[] args) {
		    new Room(7);
		    System.out.println("Hi~"); // Hi~ 다음 "방 청소"가 출력될 수도..안될 수도 있다.
    }
}
```

cleaner 의 동작을 예측할 수 없으므로 State 의 run 메소드가 실행되는 것을 보장할 수 없다.