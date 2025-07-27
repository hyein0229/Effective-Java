# Item 81. wait 와 notify 보다는 동시성 유틸리티를 애용하라
## 동시성 유틸리티
- java 5부터 도입된 것으로 wait 와 notify 가 하던 일을 대신 처리해줄 수 있다.
- **wait 와 notify는 올바르게 사용하는 것이 까다로우므로 고수준 동시성 유틸리티를 사용하라**
- `java.util.concurrent` 의 고수준 유틸리티는 세 범주로 나뉜다.
    - 실행자 프레임워크(Executor)
    - 동시성 컬렉션(concurrent collection)
    - 동기화 장치(synchronizer)

## 동시성 컬렉션(concurrent collection)
- List, Queue, Map 같은 표준 컬렉션 인터페이스에 동시성을 가미한 컬렉션이다.
- 높은 동시성에 도달하기 위해 동기화를 각자의 내부에서 수행한다.
- **따라서 동시성을 무력화하는 것은 불가능하며, 외부에서 락을 추가로 거는 것은 오히려 속도가 느려지는 일일 뿐이다.**
- 동시성 무력화하지 못하므로, 여러 메서드를 원자적으로 묶어 호출하는 것은 불가능하다.
    - 즉, 메서드의 하나하나가 원자적
    - 따라서 여러 메서드가 같이 사용될 때 그 전체를 원자적으로 만들 순 없다.
- 따라서 `상태 의존적 수정` 메서드 추가
    - 여러 기본 동작을 하나의 원자적 동작으로 묶을 수 있다. 
    - 아주 유용하여 java 8에선 일반 컬렉션 인터페이스에도 default 메서드 형태로 추가되었다.
    - ex. Map 의 putIfAbsent(key, value) 메서드:
        - 주어진 키에 매핑된 값이 없는 경우만 추가
        - 기존 값이 있다면 반환하고, 없었다면 null 을 반환한다. 
        - 이 덕에 스레드 안전한 정규화 맵을 쉽게 구현할 수 있다.

___

### ConcurrentHashMap
- ConcurrentHashMap 은 동시성이 뛰어나며 속도도 무척 빠르다.
- Collcections.synchronizedMap 보다 ConcurrentHashMap 을 사용하는 것이 훨씬 좋다.
- 동기화된 맵을 동시성 맵으로 교체하는 것만으로 동시성 애플리케이션의 성능은 크게 개선된다.
- Collcections.synchronizedMap 은 그냥 하나의 맵에 락을 걸어 한번에 하나의 스레드만 처리하도록 한 것
- ConcurrentHashMap 은 대부분 락 없이(lock-free) 읽기 가능하도록 하여 다수의 스레드가 접근할 수 있다.
```java
// ConcurrentMap 으로 구현한 동시성 정규화 맵 - 최적은 아니다.
private static final ConcurrentMap<String, String> map = new ConcurrentHashMap<>();

public static String intern(String s) {
    String previousValue = map.putIfAbsent(s, s);
    return previousValue == null ? s : previousValue;
}
```
- ConcurrentHashMap 은 get 같은 검색 기능에 최적화되었다.
- 따라서 get 을 먼저 호출하여 필요할 때만 putIfAbsent 를 호출하면 더 빠르게 개선할 수 있다.
```java
// ConcurrentMap으로 구현한 동시성 정규화 맵 - 더 빠르다!
public static String intern(String s) {
    String result = map.get(s);
    if (result == null) {
        result = map.putIfAbsent(s, s);
        if (result == null)
            result = s;
    }
    return result;
}
```
- get 으로 key 존재 확인 후, 없는 경우만 putIfAbsent 로 값을 넣는다. 
- 이 경우 key 가 존재할 가능성이 높다면 더 빨라질 것이다. get() 는 동기화 처리가 되어 있지 않다.
- putIfAbsent 만 사용한다면 내부 동기화 처리때문에 성능 차이가 발생한다. 

___

### BlockingQueue
- 컬렉션 인터페이스 중 일부는 작업이 성공적으로 완료될 때까지 기다리도록(차단되도록) 확장되었다.
- Queue 를 확장한 BlockingQueue 는 생산자-소비자 작업 큐로 쓰기에 적합하다.
    - take() 메서드는 큐의 첫 원소를 꺼내는 데 만약 큐가 비었다면, 새로운 원소가 추가될 때까지 기다린다.
    - 작업 큐는 생산자 스레드가 작업을 큐에 넣고, 소비자 스레드가 큐에 있는 작업을 처리하는 형태이다. 
    - 대부분의 실행자 서비스는 구현체(ex. ThreadPoolExecutor) 에서 BlockingQueue 를 사용한다.

## 동기화 장치(synchronizer)
- 동기화 장치 역할? 스레드가 다른 스레드를 기다릴 수 있도록 하여 서로 작업을 조율하게 해준다.
- ex. CountDownLatch, Semaphore, CyclicBarrier, Exchanger, Phaser
    - 가장 자주 쓰이는 동기화 장치는 CountDownLatch 와 Semaphore 이다.
    - 가장 강력한 동기화 장치는 Phaser 이다.

### CountDownLatch
- 일회성 장벽으로, 하나 이상의 스레드가 다른 하나의 이상의 스레드 작업이 끝날 때까지 기다리게 한다.
- latch.countDown() 는 수행 완료했으므로 카운트 해라 라는 의미이다.
- CountDownLatch 의 유일한 생성자는 int 값을 받는다.
    - 이는 래치의 countDown 메서드를 몇번 호출해야 대기 중인 스레드들을 깨우는지를 결정한다.
    - 즉, 지정한 개수의 모든 스레드의 작업이 완료될 때까지 기다리게 한다.
- 예시로 어떤 동작들을 동시에 시작해 모두 완료하기까지 시간을 재는 프레임워크를 만든다고 하자
```java
public class ConcurrentTimer {
    private ConcurrentTimer() { } // 인스턴스 생성 불가

    public static long time(Executor executor, int concurrency,
                            Runnable action) throws InterruptedException {
        CountDownLatch ready = new CountDownLatch(concurrency);
        CountDownLatch start = new CountDownLatch(1);
        CountDownLatch done  = new CountDownLatch(concurrency);

        for (int i = 0; i < concurrency; i++) {
            executor.execute(() -> {
                ready.countDown(); // 타이머에게 준비를 마쳤음을 알린다.
                try {
                    start.await(); // 모든 작업자 스레드가 준비될 때까지 기다린다.
                    action.run();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    done.countDown();  // 타이머에게 작업을 마쳤음을 알린다.
                }
            });
        }

        ready.await();     // 모든 작업자가 준비될 때까지 기다린다.
        long startNanos = System.nanoTime();
        start.countDown(); // 작업자들을 깨운다.
        done.await();      // 모든 작업자가 일을 끝마치기를 기다린다.
        return System.nanoTime() - startNanos;
    }
}
```
- 래치 3개 ready, start, done 을 사용하여 구현하였다.
- ready 래치 
    - 작업자 스레드들이 준비가 완료되었음을 타이머 스레드에 countDown() 으로 알린 후 start.await() 에서 시작 알림을 대기한다.
    - ready.await() 에선 모든 작업자 스레드가 countDown() 호출할 때까지 대기한다.
- start 래치
    - ready.await() 에서 100개의 스레드가 준비 완료를 알리면 대기가 풀리고, 시작 시간을 찍는다.
    - 그리고 start.countDown() 으로 start.await() 가 열리게 하여 모든 작업자 스레드가 다시 작업을 수행하도록 한다.
- done 래치
    - 작업자 스레드에선 본인의 일을 마치면 done.countDown(); 을 호출한다.
    - done.await() 에서 모든 100개 작업자 스레드가 일을 마칠 때까지 기다린 후 시작-끝 실행 시간을 찍는다.
- 주의점
    - time 메서드에 넘겨진 실행자(executor)는 concurrency 매개변수로 넘겨진 수만큼 스레드를 생성할 수 있어야 한다.
    - 그렇지 않으면 ready.await() 는 충분히 카운트하지 못해 이 메서드는 끝나지 않을 것이다. 
    - 이런 상태를 기아 교착상태(thread starvation deadlock)라고 한다.
    - 작업자 스레드에서 InterruptedException 을 캐치하면 스레드는 interrupt() 를 호출하여 복구하고, run() 에서 뺘져나온다.
    - 이렇게 해야 실행자가 인터럽트를 적절하게 처리할 수 있다. 
- 위 래치 3개는 CyclicBarrier(혹은 Phasser) 인스턴스 하나로 대체할 수 있다.
    - 하지만 이해하기가 더 어렵다.

**시간 간격을 잴 때는 항상 System.currentTimeMillis 가 아닌 System.nanoTime 을 사용하자**
- nanoTime 은 더 정확하고 정밀하여 시스템의 실시간 시계의 시간 보정에 영향을 받지 않는다.
- 하지만 정말 정확한 시간 측정을 하고 싶다면 jmh 같은 특수 프레임워크를 사용해야 한다.

## 레거시에서 wait 와 notify 를 사용해야 하는 경우
- 최근 코드에선 동시성 유틸리티를 사용하는 것이 맞지만 레거시 코드를 다루어야 하는 경우도 있다.
- `wait()` 메서드
    - 스레드가 어떤 조건이 충족되기를 기다리게 할 때 사용한다.
    - 락 객체의 wait 메서드는 반드시 그 객체를 잠근 동기화 영역 안에서 호출해야 한다.
```java
// wait 메소드를 사용하는 표준 방식
synchronized (obj) {
    while (<조건이 충족되지 않았다>) {
        obj.wait(); // 락을 놓고, 깨어나면 다시 잡기
    }
    ... // 조건 충족시 동작 수행
}
```

**wait() 사용 시엔 반드시 대기 반복문(wait loop)을 사용하고, 반복문 밖에선 절대로 호출하지 마라**
- 반복문은 wait() 호출 전후로 조건이 만족하는지를 검사하는 역할을 한다.
- 대기 전에 조건을 검사하여 조건이 이미 충족된 경우 wait 를 건너뛰게 하여 응답 불가 상태를 예방한다.
    - 이미 조건이 충족되었는 데도 wait() 에 들어갈 경우?
    - 스레드가 먼저 notify 를 호출한 후 wait() 로 대기 상태로 빠지면, 다시 깨울 수 있다고 보장하지 못한다.
- 대기 후에 조건을 검사하여 조건이 충족되지 않았다면 다시 대기하게 하여 안전 실패를 막는다.
    - 조건이 충족되지 않았는데 스레드가 동작을 이어가면 락이 보호하는 불변식을 깨트릴 위험이 있다.
    - 조건이 만족되지 않았는데 스레드가 깨어날 수 있는 몇 상황이 존재한다.
        - notify 호출 후 대기자가 깨어나는 사이 다른 스레드가 락을 얻어 상태를 변경하는 경우
        - 조건이 만족되지 않았는 데 다른 스레드가 실수 또는 악의적으로 notify 호출하는 경우
        - 대기 중인 스레드 중 일부만 조건이 충족되어도 notifyAll 을 호출해버리는 경우
        - 대기 중인 스레드가 (드물게) notify 없이도 깨어나는 경우, 허위 각성이라는 현상이다.

### notify vs notifyAll
- 차이는 notify 는 하나만 깨우고, notify 는 대기하는 모든 스레드를 깨운다는 것이다.
- 일반적으론 언제나 notify 를 사용하는 것이 합리적이고 안전할 것이다.
    - 깨어나야 하는 모든 스레드가 깨어남을 보장하므로 항상 정확한 결과를 얻을 것이다.
    - 깨어난 스레드는 조건이 충족되었는지 확인하고, 충족되지 않았다면 또 대기할 것이다.
- 모든 스레드가 같은 조건을 기다리며, 조건이 항 번 충족될 때마다 단 하나의 스레드만 혜택을 받을 수 있으면 notify 로 최적화할 수 있다.
- 하지만, 모든 전제 조건들이 만족하여도 notifyAll 이 나은 이유가 있다.
    - 관련 없는 스레드가 실수 또는 악의적으로 wait 를 호출하는 공격으로부터 보호될 수 있다.
    - 그러한 스레드가 중요한 notify 를 삼켜버리면 꼭 깨어났어야 하는 스레드들이 계속 대기하게 될 수 있다.
    - 즉, notify 는 어떤 스레드가 깨어날 지를 보장할 수 없으므로 안전하게 notifyAll 을 사용해라

## 요약 정리 
- 새로운 코드를 작성한다면 wait 와 notify 대신 고수준의 동시성 유틸리티(java.util.concurrent)를 사용해라
- 만약 레거시 코드에서 wait 를 사용하게 된다면 반드시 while 문 안에서 호출하도록 하자
- 일반적으론 notify 보단 notifyAll 을 사용해야 한다.
    - 혹시나 notify 를 사용한다면 응답 불가 상태에 빠지는 것을 주의해야 한다.
