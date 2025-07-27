# Item 82. 스레드 안전성 수준을 문서화하라
## API 문서에서의 synchronized 한정자
- javaDoc 이 기본 옵션으로 생성한 API 문서에는 synchronized 한정자가 포함되지 않는다.
    - synchronized 는 메서드 선언 시 구현 이슈일 뿐 API 에 속하지 않는다.
- 따라서 synchronized 만으로는 메서드가 스레드 안전하다고 믿기 어렵다.
- 내부 동작에 따라 그 안전성이 보장되지 않을 수 있고, 안전성의 수준도 다르다.

## 스레드 안전성 수준 문서화
- 스레드 안전성에도 수준이 나뉜다.
- 멀티스레드 환경에서도 API를 안전하게 사용하기 위해선 클래스가 지원하는 안전성 수준을 명시해야 한다.
- 스레드 안전성 종류
    - 불변(immutable): 불변이므로 상태가 바뀔 일이 없다. 즉, 동기화가 전혀 필요 없다.
    - 무조건적 스레드 안전(unconditionally thread-safe):
        - 이 클래스 인스턴스는 수정될 수 있으나, 내부에서 동기화하여 별도의 외부 동기화 없어도 안전하다.
        - AtomicLong, ConcurrentHashMap 이 여기에 속한다. 이는 복합 연산까지 원자적 수행 메서드로 제공한다.
    - 조건부 스레드 안전(conditionally thread-safe):
        - 일부 메서드는 동시에 사용하려면 외부 동기화가 필요하다.
        - Collections.synchronize 래퍼 메서드가 반환한 컬렉션들이 여기에 속한다.
    - 스레드 안전하지 않음(not thread-safe):
        - 인스턴스가 수정될 수 있고, 동시에 사용하려면 각각의 메서드 호출을 외부 동기화 메커니즘으로 감싸야 한다.
        - ArrayList, HashMap 등 기본 컬렉션이 속한다.
    - 스레드 적대적(thread-hostile):
        - 모든 메서드 호출을 외부 동기화로 감싸도 멀티스레드 환경에서 안전하지 않다. 
        - 스레드 적대적으로 밝혀진 클래스나 메서드는 일반적으로 deprecated API 가 되었다.

### 조건부 스레드 안전한 클래스는 주의해서 문서화해라
- 어떤 순서로 호출될 때 외부 동기화가 필요한지,
- 그 순서로 호출하기 위해선 어떤 락을 얻어야 하는지 알려줘야 한다.
- 일반적으로는 해당 인스턴스 잧체를 락으로 얻지만 예외도 존재한다.

## 비공개 락 객체를 사용해라
만약, 클래스가 외부에서 사용할 수 있는 락을 제공한다면?
```java
public class SharedLock {
    public final Object lock = new Object(); // public 으로 공개된 락

    public void doSomething() {
        synchronized(lock) {
            // 어떤 작업 수행
        }
    }
}
```
```java
public class Client {
    public void doSomethingWithLock(SharedLock sharedLock) {
        synchronized(sharedLock.lock) {
            // 여러 메서드를 클라이언트가 원자적 수행
        }
    }
}
```
- 장점: 클라이언트가 일련의 메서드 호출을 원자적으로 수행할 수 있도록 유연성 제공
- 단점
    - 내부에서 처리하는 고성능 동시성 제어 메커니즘과 혼용할 수 없다.
    - 따라서 ConcurrentHashMap 과 같은 동시성 유틸리티와 함께 사용할 수 없다.
    - 클라이언트가 오래 락을 쥐고 놓지 않으면 서비스 거부 공격(dos)을 수행할 수도 있다.

**공격 가능성을 막기 위해선 비공개 lock 객체를 사용해라**
```java
private final Object lock = new Object(); // 클래스 내부에서만 락 객체를 사용하도록 함

public void foo() {
    synchronized(lock) {
        // ...
    }
}
```
- 락 객체가 외부 노출이 안되므로 클라이언트가 동기화할 수 없다.
- **락 필드는 항상 final 로 선언해라**
    - final 선언하여 락 객체가 교체되는 일을 방지한다.
    - 일반적인 락이든 java.util.concurrent.locks 패키지 락이든 마찬가지로 final 선언해야 한다.
- 비공개 락 객체 관용구는 무조건적 스레드 안전 클래스에서만 사용할 수 있다.
    - 조건부 안전 클래스는 필요한 락을 잡아 외부 동기화 처리할 수 있어야 한다. 따라서 락을 숨기면 안된다.
- 비공개 락 객체는 상속용으로 설계한 클래스에 특히 잘 맞는다.
    - 상위 클래스와 하위 클래스가 자신 인스턴스(this)를 락으로 잡아 공유하는 상태가 되면 동작에 문제가 발생할 수 있다.
    - 실제로 Thread 클래스에서 나타나는 문제이다.
    - Thread 클래스의 일부 메서드는 synchronized(this) 로 동기화되므로 확장할 때 문제가 발생할 수 있다.
    
## 요약 정리
- 모든 클래스가 자신의 스레드 안전성 정보에 대해 문서화해야 한다.
- 정확한 언어 설명이나 스레드 안전성 어노테이션을 사용할 수 있다.
- synchronized 한정자는 문서화와는 관련이 없다. 
- 조건부 스레드 안전 클래스는 메서드를 어떤 순서로 호출했을 때 외부 동기화가 필요한지 알려줘야 한다.
- 무조건적 스레드 안전 클래스를 작성할 땐 private final 락 객체를 선언하고 사용하자.