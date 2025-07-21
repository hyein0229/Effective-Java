# Item 79. 과도한 동기화는 피하라
- 과도한 동기화는 성능을 떨어트리고, 교착상태에 빠뜨릴 수 있다.
- 또한 예측할 수 없는 동작을 낳기도 한다.

## 동기화 블록 안에선 제어를 클라이언트에게 넘기지 마라
- 동기화된 영역 안에선 재정의할 수 있는 메서드나 클라이어트가 넘겨준 함수 객체를 호출해선 안된다.
- 이를 외부에서 들어온 **외계인 메서드(alien method)** 이다.
- 외계인 메서드는 하는 일에 따라 예외를 일으키거나 교착상태에 빠지게 하거나, 데이터를 훼손할 수 있다.
<br>

### `안전 실패(safety failure) 문제`
```java
// Set 을 감싸는 래퍼 클래인 ObservableSet 은 관찰자 패턴을 구현하였다.
// 이 클래스의 클라이언트는 set 에 원소가 추가되면 알림을 받을 수 있다. 
public class ObservableSet<E> extends ForwardingSet<E> {

    public ObservableSet(Set<E> set) {
        super(set);
    }

    private final List<SetObserver<E>> observers = new ArrayList<>();

    public void addObserver(SetObserver<E> observer) {
        synchronized (observers) {
            observers.add(observer);
        }
    }
    
    public boolean removeObserver(SetObserver<E> observer) {
        synchronized (observers) {
            return observers.remove(observer);
        }
    }
    
    // 관찰자 리스트를 순회하며 observer.added() 호출 (외계인 메서드)
    private void notifyElementAdded(E element) {
        synchronized (observers) {
            for(SetObserver<E> observer : observers) {
                observer.added(this, element); // 콜백 실행
            }
        }
    }
    
    @Override
    public boolean add(E element) {
        boolean added = super.add(element);
        if(added) {
            notifyElementAdded(element); // 원소 추가 시 알림 메서드 호출
        }
        return added;
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        boolean result = false;
        for (E element : c) {
            result |= add(element); //notifyElementAdded를 호출
        }
        return result;
    }
}
```
- add 메서드 내부에서 notifyElementAdded() 메서드를 호출한다.
- notifyElementAdded 에서 관찰자 리스트를 순회하며 각 관찰자에게 알림을 보낸다.
```java
@FunctionalInterface
public interface SetObserver<E> {
    // ObservableSet에 원소가 추가되면 호출된다.
    void added(ObservableSet<E> set, E element);
}
```
- 이 함수형 인터페이스를 구현하여 관찰자 객체를 만들어낸다.
- ObservableSet 에서 원소가 추가될 때 이 added 메서드가 호출된다.
- BiConsumer<T, U> 와 유사하지만 이름이 더 직관적이고 다중 콜백을 지원하도록 확장할 수 있다.

```java
public static void main(String[] args) {
	ObservableSet<Integer> set = new ObservableSet<>(New HashSet<>());
	
	set.addObserver(new SetObserver<Integer>() {
		public void added(ObservableSet<Integer> s, Integer e) {
			System.out.println(e); 
			if (e == 23) // 원소가 23이면 관찰자 자신을 리스트에서 제거하도록 한다. 
                s.removeObserver(this);
		}
	});

	for (int i = 0; i < 100; i++) 
		set.add(i);
}
```
- 0부터 23까지 출력한 후 관찰자 자신이 구독 해지된 후 종료될 것 같지만 그렇지 않다.
- 이 프로그램은 23까지 출력한 다음 **ConcurrentModifiedException** 을 던진다.
- 관찰자 list 를 순회화며 관찰자의 added 호출 -> added 내부에서 자신을 리스트에서 제거하는 removeObserver 호출한다.
    - 즉, 리스트를 순회 중에 리스트에서 원소를 제거하려하니 허용되지 않은 동작을 한 것이다.
- notifyElementAdded 메서드에서 수행하는 순회는 synchronized 블록 안에 있으므로 동시 수정이 일어나지 않도록 보장한다. 하지만, 정작 자기 자신(같은 스레드)이 콜백을 거쳐 수정하는 것까진 막지 못한다.

### `응답 불가(교착 상태 문제)`
```java
set.addObserver(new SetObserver<Integer>() {
    public void added(ObservableSet<Integer> s, Integer e) {
        System.out.println(e);
        if (e == 23) {
            ExecutorService exec = Executors.newSingleThreadExecutor();
            try {
                exec.submit(() -> s.removeObserver(this)).get();
            } catch (ExecutionException | InterruptedException ex) {
                throw new AssertionError(ex);
            } finally {
                exec.shutdown();
            }
        }
    }
});
```
- 구독해지 할 때 removeObserver 를 직접 호출하지 않고 실행자 서비스(ExecutorService) 를 사용하여 다른 스레드에 부탁한다.
- 예외는 발생하지 않지만 교착상태에 빠진다.
- 백그라운드 스레드가 removeObserver 를 호출하면 observers 를 잠그려고 시도하지만 락을 얻을 수 없다.
- 메인 스레드가 observers 에 대한 락을 이미 갖고 있기 때문이다.
- 동시에 메인 스레드는 백그라운드 스레드의 작업이 수행되기만을 기다린다.
- 즉, 하나는 락을 얻기 위해 기다리고 하나는 락을 쥐고 상대의 작업을 기다리므로 데드락(deadlock)에 빠진다.


## java 언어의 락 - `재진입(reentrant)`
- 자바에서 락은 재진입(reentrant)을 허용하므로 교착 상태에 빠지지는 않는다.
    - 자기 자신이 synchronized 블록을 호출하여 락을 얻음 -> 내부에서 또 synchronized 를 만나면?
    - 재진입이 불가하다면 본인이 락을 쥐고 있으므로 락을 요청하면 얻을 수 없어 데드락 발생
    - 재진입은 본인이 락을 들고 있어도 또 락을 획득할 수 있다는 뜻
- 즉, 락을 쥐고 있는 스레드가 다음번 락 획득도 성공할 수 있다.
- 문제는 그 락이 보호하는 데이터와는 관련이 전혀 없는 작업이 진행 중일 수 있다는 것이다.
    - 이 경우 참혹한 결과가 빚어질 수 있다.
    - 원인은 락이 제 구실을 제대로 하지 못했기 때문이다. 불필요하게 락을 길게 보유한다.
- 재진입 가능 락은 객체 지향 멀티스레드 프로그램을 쉽게 구현하도록 돕지만, 응답 불가가 될 상황을 안전 실패(데이터 실패)로 변모시킬 수 있다. 

___

### 해결1 - 외계인 메서드 호출을 동기화 블록 바깥으로 옮기기
```java
private void notifyElementAdded《E element) { 
    List<SetObserver<E» snapshot=null; 
    synchronized(observers) { 
        snapshot = new ArrayList<>《observers); 
    }
    for (SetObserver<E> observer : snapshot) 
        observer.added(this, element);
}
```
- 관찰자 리스트를 복사하여 사용하여 락 없이 원본 리스트는 지키면서 안전하게 순회하도록 한다.
- 이 방식은 예외 발생과 교착 상태 문제를 모두 해결할 수 있다.
- 외계인 메서드 호출을 동기화 블록 바깥으로 옮기는 더 나은 방법엔 CopyOnWriteArrayList 가 있다.


### 해결2 - 자바 동시성 컬렉션 라이브러리 CopyOnWriteArrayList 사용
```java
private final List<SetObserser<E>> observers = new CopyOnWriteArrayList<>(); 

public void addObserver(SetObserver<E> observer) { 
    observers.add(observer);
}

public boolean removeObserver(SetObserver<E> observer) { 
    return observers.remove(observer);
}

public void notifyElementAdded(E element) { 
    for (SetObserver<E> observer : observers) {
         observers.added(this, element);
    }
}
```
- CopyOnWriteArrayList 는 내부를 변경하는 작업은 항상 복사본을 만들어 수행하도록 구현되었다.
- 내부 배열은 절대 수정하지 않으므로 순회할 때 락이 필요 없어 매우 빠르다.
- 수정할 일은 드물고 순회만 빈번히 일어나는 관찰자 리스트 용도에는 최적이다.

## 열린 호출(open call)
- 동기화 영역 바깥에서 호출되는 외계인 메서드 호출을 말한다.
- 외계인 메서드는 얼마나 오래 실행될지 알 수 없으므로, 동기화 영역 안에서 호출하면 다른 스레드는 계속 대기해야만 한다.
- 열린 호출은 **실패 방지 효과와 동시성 효율을 크게 개선한다.**
- **기본적으로 동기화 영역에선 가능한 한 일을 적게 수행해야 한다.**
- 따라서 오래 걸리는 작업이라면 가능하다면 동기화 영역 바깥으로 옮겨 처리하는 것이 좋다.

## 동기화의 성능 문제
- 자바의 동기화 비용은 계속 낮아져 왔지만, 과도한 동기화는 역시 피해야 한다.
- 오늘날 멀티코어 환경에서 동기화가 초래하는 진짜 비용은 락을 얻는 데 드는 CPU 시간이 아니다.
- **병렬로 실행할 기회를 읽고, 모든 코어가 일관된 메모리 상태를 보기 위해 필요한 지연 시간이 주요 비용이다.**
- 또한 동기화는 jvm 상에서의 코드 최적화를 제한한다는 단점도 있다.

## 가변 클래스를 작성할 때의 두 가지 선택지
- 첫 번째, 내부 동기화는 전혀 하지 말고 그 클래스를 사용하는 외부에서 동기화하는 것이다.
    - ex. java.util(Vector, Hashtable 제외), StringBuilder, java.util.concurrent.ThreadLocalRandom
- 두 번째, 동기화를 내부에서 수행하여 스레드 안전 클래스로 만든다.
    - 단, 외부에서 객체 전체에 락을 거는 것보다 동시성을 월등히 개선할 수 있는 경우만 선택해야 한다.
    - ex. java.util.concurrent, StringBuffer, java.util.Random
    - 락 분할, 락 스트라이핑, 비차단 동시성 제어 등 다양한 기법을 동원하여 동시성을 높여줄 수 있다.
    - 여러 스레드가 호출할 가능성이 있는 메서드가 정적 필드를 수정한다면 반드시 동기화해야 한다.
        - ex. item 78 에서의 generateSerialNumber 메서드의 nextSerialNumber 필드
- 선택이 어렵다면 동기화하지 말고, 문서에 "스레드 안전하지 않다" 라고 기록하자

## 요약 정리
- 교착 상태와 안전 실패(데이터 훼손)을 피하려면 동기화 영역 안에서 외계인 메서드를 호출해선 안된다.
- 즉, 동기화 영역 안에서의 작업은 최소한으로 줄여라
- 가변 클래스 설계 시엔 스스로 동기화(내부 동기화)해야 할 지를 고민해라
    - 동기화에 대한 여부는 문서화해야 한다.
- 멀티코어 세상인 지금은 과도한 동기화로 인해 성능 저하를 피하는 것이 과거보다 중요하다.
