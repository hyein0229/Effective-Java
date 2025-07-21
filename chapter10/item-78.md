# Item 78. 공유 중인 가변 데이터는 동기화해 사용하라

## 동기화의 기능
- 원자성(배타적 실행)
    - synchronized 키워드는 해당 메서드나 블록을 한번에 한 스레드씩만 수행하도록 보장한다.
    - 한 객체가 일관된 상태를 가지고 생성되고, 이 객체에 접근하는 메서드는 락(lock)을 건다.
    - 락을 건 메서드는 객체의 상태를 확인하고 수정하여, 객체를 하나의 일관된 상태에서 다른 일관된 상태로 변화시킨다.
    - 따라서 상태가 일관되지 않는 순간의 객체를 다른 스레드가 접근하거나 변경하는 것을 막는다.
    - 즉, 항상 일관된 상태를 보게 하도록 보장한다.
- 가시성
    - 한 스레드가 변경한 값을 다른 스레드에서 볼 수 있는 특성이다.
    - 자바에선 스레드마다 작업 메모리(캐시)를 가지고 연산을 수행하므로 다른 스레드에서 변경한 값을 바로 보지 못할 수 있다.
    - 동기화를 하면 한 스레드가 변경한 값이 즉시 메인 메모리에 반영되고, 읽을 땐 메인 메모리에서 가져오도록 하여 언제나 최종 수정 결과를 보장하게 한다.
    - 따라서 스레드 간의 데이터 일관성을 보장하게 한다.

___

### long 과 double 외의 변수를 읽고 쓰는(할당) 동작은 원자적(atomic)이다.
- 즉, 원자적이라는 것은 배타적 실행이 이미 보장되어 다른 스레드가 중간에 끼어들 수 없다는 것이다.
- 여러 스레드가 같은 변수를 동기화 없이 수정해도, 항상 어떤 스레드가 저장한 값을 온전히 읽어올 수 있다.
- long 이나 double 과 같은 64비트 데이터는 32비트씩 쪼개어 처리한다. 
    - 하지만, 이건 jvm 의 기본 처리 단위에 따라 다르다.
    - 최근 64비트 jvm 에선 64비트로 비원자적 접근을 하지 않고 처리할 수 있다고 한다.
- **하지만 가시성은 보장하지 않는다.** 
    - 언제 다른 스레드가 수정한 값을 보게될 지는 보장하지 못한다.
    - 따라서 **동기화는 배타적 실행뿐 아니라 스레드 사이의 안정적 통신에 필요하다.**
    - 이는 한 스레드가 만든 변화가 다른 스레드에게 언제 어떻게 보이는지를 규정한 자바의 메모리 모델 때문이다.

## Thread.stop 은 사용하지 마라
- 이 메서드는 안전하지 않아 이미 오래전에 deprecated 된 API 이다.
- 스레드가 작업을 온전히 끝내지 않고 멈출 수 있어 데이터가 훼손될 수 있다.   

## 대신 스레드를 올바르게 멈추는 방법 - `flag 폴링 방식`
```java
public class StopThread {
	private static boolean stopRequested;
    
    public static void main(String[] args) throws InterruptedException {
    	Thread th = new Thread(() -> {
        	int i = 0;
            while (!stopRequested) {
            	i++;
                System.out.println(i);
            }
        });
        th.start();
        
        TimeUnit.SECONDS.sleep(1);
        stopRequested = true;
    }
}
```
- 첫 번째 스레드는 자신의 boolean 필드를 폴링하면서 true 가 되면 멈춘다.
- 해당 필드를 false 로 초기화한 후 다른 스레드가 스레드를 멈추고 싶을 때 true 로 변경하는 것이다.
- boolean 필드를 읽고 쓰는 것은 원자적이다.
- 하지만 위 코드는 의도대로 1초 후에 종료될까? 
    - 결과는 반복문을 빠져나오지 못하고 영원히 수행되었다.
- 원인1 - 가시성
    - 동기화를 하지 않으면 메인 스레드가 수정한 값을 백그라운드스레드가 언제 볼지 보증할 수 없다.
    - 스레드는 작업 메모리(캐시)를 사용한다고 했다. 메인 메모리에서 변경된 값이 즉시 반영안될 수 있다.
- 원인2 - 동기화를 하지 않으면 끌어올리기(hoisting)라는 최적화 기법이 수행될 수 있다.
    - 이로 인해 **응답 불가(livenes failure)** 상태가 된다.
    ```java
    //원래 코드
    while(!stopRequested) {
        i++;
    }

    // 최적화한 코드
    if(!stopRequested) {
        while(true) {
            i++;
        }
    }
    ```

## `synchronized` 키워드로 동기화
```java
public class StopThread {

    private static boolean stopRequested;
    
    private static synchronized void requestStop() { 
        stopRequested = true;
    }
    
    private static synchronized boolean stopRequested() { 연산을 수행
        return stopRequested;
    }

    public static void main(String[] args) throws InterruptedException {

        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while(!stopRequested()) {
                i++;
            }
        });

        backgroundThread.start();

        TimeUnit.SECONDS.sleep(1);
        stopRequested();
    }
}
```
- 쓰기 메서드(requestStop)와 읽기 메서드(stopRequested) 모두 각각 동기화해야 한다.
- 쓰기와 읽기 모두가 동기화되지 못하면 동작을 보장하지 못한다.
- 여기서 synchronized 동기화는 순전히 동기화 기능 중 **통신 목적(가시성)** 으로만 사용되었다.
    - 사실 각 메서드는 동기화를 하지 않아도 원자적으로 동작한다.

## `volatile` 동기화
- volatile 한정자는 배타적 수행은 보장하지 않지만 가시성을 보장한다.
- 값을 수정하면 메인 메모리에 즉시 쓰고, 읽을 때도 메인 메모리에서 읽도록 한다.
- 즉, 항상 가장 최근에 기록된 값을 읽게 됨을 보장할 수 있어 다른 스레드가 수정한 값을 즉시 알 수 있다.
- 배타적 실행은 보장하지 않으므로 synchronized 보다 속도가 더 빠르다. 
```java
public class StopThread {

    private static volatile boolean stopRequested; // volatile 변수로 선언

    public static void main(String[] args) throws InterruptedException {

        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while(!stopRequested) {
                i++;
            }
        });

        backgroundThread.start();

        TimeUnit.SECONDS.sleep(1);
        stopRequested = true;
    }
}
```

___

### volatile 주의점
```java
private static volatile int nextSerialNumber = 0;

public static int generateSerialNumber() {
    return nextSerialNumber++; // 산술 연산자 사용
}
```
- 위 코드는 멀티스레딩 환경에서 동기화 없이는 제대로 동작하지 못한다.
- 증가 연산자(++)는 실제로는 필드에 두 번 접근한다. (원자적으로 수행되지 않는다.)
    - 먼저 값을 읽고, 다음 새로운 값을 저장한다.
    - 스레드가 값을 증가시키기 전에 다른 스레드가 값을 읽어가면 변경되기 전 상태의 값을 읽어가서 수행한다.
    - 따라서 최종 결과가 잘못된 결과가 나올 수 있다. 이런 오류를 **안전 실패(safety failure)** 라고 한다.
- synchronized 락 동기화를 사용하여 해결할 수 있다.

## `java.util.concurrent.atomic` 동기화
- 락 없이도(lock-free) 스레드 안전한 프로그래밍을 지원하도록 설계된 클래스들이 있다.
- 이 패키지는 원자성(배타적 실행)과 가시성을 모두 지원한다.
- synchronized 는 동기화 비용이 크므로 그보다 성능이 우수하다.
```java
private static final AtomicLong nextNum = new AtomicLong();

public static long generateNumber() {
	return nextNum.getAndIncrement();
}
```

## 참고
- 가장 좋은 방법은 데이터를 공유하지 않는 것이다.
- 불변 데이터만 공유하거나 아무것도 공유하지 말자
    - 즉, 가변데이터는 단일 스레드 내에서만 사용하도록 하자
    - 그리고 이 정책을 문서화하여라
- 사용하는 프레임워크나 라이브버리를 깊이 이해하자
    - 우리가 인지하지 못한 스레드를 수행하는 복병으로 작용하는 경우가 있다.
- 하나의 객체에서 공유되는 부분만 동기화해도 된다.
    - 또 수정되기 전에는 동기화 없이 다른 스레드에서 자유롭게 읽을 수 있다.
    - 이러한 객체를 공유하는 행위를 안전 발행(safe publication)이라 한다.
- 객체를 안전 발행하는 방법
    - 정적 필드
    - volatile 필드
    - final 필드
    - 락을 통해 접근하는 필드
    - 동시성 컬렉션(ex. ConcurrentHashMap)

## 요약 정리
- 여러 스레드가 가변 데이터를 공유한다면 그 데이터를 읽고 쓰는 동작은 반드시 동기화해야 한다.
- 동기화하지 않으면 한 스레드가 한 변경을 다른 스레드에서 보지 못할 수 있다.
- 동기화를 실패하면 응답 불가 상태나 안전 실패로 이어질 수 있다.
- 배타적 실행 필요 없이 가시성(스레드간 통신)만 필요하다면 volatile 만 사용할 수 있다.