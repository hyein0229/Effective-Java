# Item 07. 다 쓴 객체 참조를 해제하라

## 왜 참조를 해제해야 할까?

**GC 는 더 이상 참조 하지 않아 접근 불가능한 객체, 즉 Unreachable 객체를 GC 의 수거 대상으로 인식하여 청소를 한다.** 따라서 더 이상 사용하지 않는 객체는 참조를 해제해주어야 정상적으로 GC 에 의해 제거될 수 있다. 제거되지 않으면 더 이상 사용되지 않는 객체들이 쌓여 프로그램 성능에 영향을 주는 `메모리 누수 (Memory Leak)` 문제가 발생할 수 있다.

따라서, GC 만 믿고 메모리 관리를 아예 신경쓰면 안된다!!! 

## 메모리 누수가 발생하는 상황

### 1. 자기 메모리를 직접 관리하는 클래스

```java
public class Stack {
    private Object[] elements; // 참조 객체 배열
    private int size = 0; // 활성 영역 크기
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }
		
		// 여기서 활성 영역을 줄여줌
    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        return elements[--size]; 
    }

    /**
     * 원소를 위한 공간을 적어도 하나 이상 확보한다.
     * 배열 크기를 늘려야 할 때마다 대략 두 배씩 늘린다.
     */
    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

위 스택 클래스는 elements 배열로 저장소 풀을 만들어서 직접 원소들을 관리하고 있다.

**메모리 누수가 발생하는 지점**

- pop 메소드 호출로 인해 꺼내진 객체들을 GC 가 회수할 수 없음
- elements 배열에서 여전히 객체들의 참조를 가지고 있기 때문
- 즉, 활성 영역(size) 밖의 참조들이 더 이상 사용되진 않지만 참조되고 있어 GC 의 수거 대상이 아님

이 스택 프로그램을 오래 실행하면 결국 GC 활동과 메모리 사용량이 늘어나 성능에 영향을 줄 것이다.

가비지 컬렉션 언어에선 의도치 않게 객체를 살려둠으로써 메모리 누수가 발생하는 부분을 찾기가 매우 어렵다. 객체 하나를 살려둠으로 인해서 해당 객체가 참조하고 모든 객체들 (연쇄적) 을 모두 회수할 수 없으므로 소수의 객체 → 매우 많은 객체를 회수할 수 없도록 하여 성능에 큰 악영향을 줄 수 있다.


**`다 쓴 참조 객체는 null 처리??`**

참조를 다 쓴 객체는 null 처리하여 참조 해제하면 메모리 누수 문제가 해결된다.

```java
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();
    Object result = elements[--size]; // 반환할 객체
    elements[size] = null; // 다 쓴 참조 해제 
    return result;
}
```

null 처리로 인해서 메모리 누수를 막을 수 있고, 또한 null 처리한 객체를 잘못 참조하려는 경우 NPE 을 던지며 종료되기 때문에 잘못된 접근으로 인한 잘못된 동작을 막을 수 있다. (오류 조기 발견)

**하지만!! 객체 참조를 null 처리하는 일은 예외적이어야 한다.**

위와 같은 Stack 클래스는 elements 배열을 통해 자기 메모리를 직접 관리 중이다. 따라서 GC 에선 어디가 활성 영역이고 비활성 영역인지를 알 길이 없고 프로그래머만이 알 수 있다. 따라서 프로그래머가 명시적으로 null 처리하여 GC 에 수거 대상임을 알려야 한다.

하지만, 더 이상 필요 없어진 객체들을 모두 쓰자마자 다 null 처리하는 것은 프로그램을 필요 이상으로 지저분하게 만든다. **다 쓴 참조를 해제하는 가장 좋은 방법은 해당 참조를 담은 변수의 scope 를 최소화하여 scope 밖으로 밀어내는 것이다.**

```java
Object pop() {

	Object e = 24;

	...

	e = null; // 명시적 null 처리하지 않아도 pop 메소드가 끝나면 자동 참조 해제
}
```

Object e 는 pop 메소드 내부에서의 지역 변수이므로 pop 메소드 호출이 끝나면 참조 하고 있는 변수 e 가 제거되므로 참조가 해제되어 GC 의 수거 대상이 된다. 따라서  웬만하면 명시적 null 처리보다 scope 를 이용하는 방법이 더 좋다.

___

### 캐시

객체 참조를 캐시에 넣은 후 객체를 다 쓴 이후로도 캐시를 비우지 않으면 메모리 누수를 일으킬 수 있다. 캐시 외부에서 키를 참조하는 동안만 엔트리가 살아 있는 캐시가 필요하다면 다 쓴 엔트리는 자동으로 제거해주는 WeakHashMap 을 사용하여 캐시를 만들 수 있다. 

```java
public class CacheSample {
	public static void main(String[] args) {
		Object key = new Object();
		Object value = new Object();
		
		Map<Object, List> cache = new WeakHashMap<>();
		cache.put(key, value);
		
		key = null; // key 참조를 제거 -> cache 에서 GC 이후 엔트리가 자동 제거
		
		...
	}
}
```

WeakHashMap 은 key 를 모두 WeakReference 로 감싸서 참조 해제된 key 의 엔트리를 자동으로 삭제하도록 한다. 


❓ **자바의 참조 유형 (Strong, Soft, Weak, Phantom)**

1. Strong Reference (강한 참조)

보통 우리가 new 로 인스턴스를 만들 때의 참조 유형이다.

```java
StringBuffer buffer = new StringBuffer();
```

 참조가 해제되면 GC 의 수거 대상이 “될 수 있다”, 즉 GC 가 수거 대상으로 인식하여 제거될 수 있으며 반드시 수거된다는 보장은 없다.

2. Soft Reference (소프트 참조)
Strong 레퍼런스를 Soft 레퍼런스로 감싸 생성하며 get() 으로 얻을 수 있다.

Weak 과의 차이는 Weak 은 다음 GC 에 무조건 수거되지만 Soft 의 경우 메모리량에 따라 다르다. JVM 여유 메모리가 부족한 경우에만 삭제되면 부족하지 않다면 굳이 삭제하지 않는다.

3. Weak Reference (약한 참조)

Strong 레퍼런스를 Weak 레퍼런스로 감싸 생성하며 Weak 레퍼런스에서 get() 로 얻을 수 있다.

```java
WeakReference<Widget> weakWidget = new WeakReference<Widget>(widget);
```

이때, Strong 레퍼런스만 참조가 해제되어도 GC 의 수거 대상이 되어 다음 GC 타이밍에 힙 메모리에서 **반드시 GC 로부터 제거된다.** 따라서 “수거하라” 명령하는 것과 같다.

4. Phantom Reference (팬텀 참조)
Reference Queue 는 garbage 로 판단되는 Object 가 넣어지는 큐로써 Object 의 상태를 트래킹해볼 수 있는데, Weak Reference 의 경우엔 객체가 weakly reachable 이라고 판단 될 때 enqueue 된다. 따라서, GC 의 수거 대상으로 인식되는 상태지만 아직은 finalize 와 GC 가 일어나지 않은 시점이다.

Phantom Reference 는 무조건 생성자에서 Reference Queue 가 필요하며 get() 사용 시 null 이 리턴된다. Weak 과는 다르게 finalize 호출 후, 즉 객체가 GC에 의해 수거되기 직전에 enqueue 되며 보통 해당 객체가 죽었는지, 명확하게 메모리에서 제거되었는지 확인하기 위해 사용한다고 한다.


캐시를 만들어 사용할 때 캐시 엔트리의 유효 기간을 정확히 정의하기 어려우므로 시간이 지날수록 엔트리의 가치를 떨어뜨리는 방식을 흔히 사용하는데 이 경우에도 쓰지 않는 엔트리를 제거해주는 것이 필요하다.

ScheduledThreadPoolExecutor 와 같은 백그라운드 스레드를 활용하거나 캐시에 엔트리 추가 시에 부수 작업으로 수행하는 방법이 있다. LinkedHashMap은 removeEldestEntry 메소드를 사용하여 후자의 방식으로 처리한다. 
 
___

### 리스너, 콜백

클라이언트가 콜백을 등록만 하고 명확히 해지하지 않는 경우 콜백은 계속 쌓여갈 수 있다. 따라서 콜백을 WeakHashMap 에 key 로 등록하여 Weak Reference 로 저장하도록 하면 GC 가 즉시 수거할 수 있도록 조치할 수 있다.

콜백과 리스너가 정확히 무엇이냐

**`콜백이란?`**

콜백 메소드는 다른 함수의 인수로 전달되는 함수로, 어느 이벤트 후에 실행된다. 콜백 사용의 용도는 제어권을 넘긴 코드에서 어떤 작업이 끝난 후 반대로 콜백을 호출하여 작업 완료를 알리고 후 작업을 처리하도록 하는 것이다.

**`리스너란?`**

특정 이벤트를 처리하는 인터페이스로 어떠한 이벤트가 발생하였을 때 해당 이벤트에 맞는 처리를 한다.

**콜백과 리스너의 차이점**

- 콜백은 이벤트 발생 후 특정 함수를 호출하여 후속 처리하므로 보통 1회성 호출
- 리스너는 반복되는 이벤트를 처리하기 위함, n 번의 호출