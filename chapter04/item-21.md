# Item 21. 인터페이스는 구현하는 쪽을 생각해 설계하라

- 자바 8 이전
    - 구현체를 깨트리지 않고는 인터페이스에 메소드를 추가할 수 없다.
    - 인터페이스에 메소드 추가하는 순간 구현체에서 정의하지 않으면 메소드를 컴파일 오류가 발생한다.
    - 따라서, ***“현재의 모든 인터페이스가 새로운 메소드가 추가될 일은 영원히 없다”* 는 가정하에 작성되었다.**
- 자바 8 이후
    - **default 메소드를 정의**할 수 있다.
    - 그러나, 이전부터 존재하던 **기존 구현체들과 매끄럽게 연동되리라는 것을 보장할 수 없다.**
    - 디폴트 메소드는 구현 클래스에 대해 모른 채 합의 없이 무작정 삽입될 뿐이다.

## 디폴트 메소드 작성은 어렵다.

자바 8에선 핵심 컬렉션 인터페이스들에 다수의 디폴트 메소드가 추가되었다. 주로 람다를 활용하기 위함이다. 하지만, 생각할 수 있는 **모든 상황에서 불변식을 해치지 않는 디폴트 메소드를 작성하기란 어렵다.**

자바 8의 Collection 인터페이스에 추가된 removeIf 디폴트 메소드를 예로 생각해보자. 

```java
default boolean removeIf(Predicate<? super E> filter) {
        Objects.requireNonNull(filter);
        boolean removed = false;
        final Iterator<E> each = iterator();
        while (each.hasNext()) {
            if (filter.test(each.next())) { // predicate 가 true 인 원소 모두 제거
                each.remove();
                removed = true;
            }
        }
        return removed;
    }
    
list.removeIf(number -> number.equals("1")); // removeIf 사용 예
```

이 디폴트 메소드는 현존하는 모든 Collection 구현체와 잘 어우러지는 것은 아니다. 

예로 아파치 커먼즈 라이브러리의 SynchronizedCollection 클래스는 클라이언트가 제공한 객체로 락을 거는 능력을 추가로 제공한다. 즉, 모든 메소드에서 주어진 락 객체로 동기화한 후 내부 컬렉션 객체에 기능을 위임하는 래퍼 클래스이다.

책 작성 시점엔 **removeIf 메소드가 재정의되어 있지 않아 대신 디폴트 메소드가 호출되게 되고, 모든 메소드 호출을 알아서 동기화해주지 못하는 문제가 일어날 수 있다.** 디폴트 메소드는 동기화에 대해 전혀 모른다. 따라서 여러 스레드가 동시 접근 시 ConcurrentModificationException을 발생시키거나, 예기치 못한 결과로 이어질 수 있다.

그러나, 현재는 removeIf 메소드가 재정의된 것으로 보인다.

```java
// https://mvnrepository.com/artifact/org.apache.commons/commons-collections4/4.4
public class SynchronizedCollection<E> implements Collection<E>, Serializable {

    /** Serialization version */
    private static final long serialVersionUID = 2412805092710877986L;

    /** The collection to decorate */
    private final Collection<E> collection;
    /** The object to lock on, needed for List/SortedSet views */
    protected final Object lock;
		
		...

	@Override
    public boolean remove(final Object object) {
        synchronized (lock) {
            return decorated().remove(object);
        }
    }

    /**
     * @since 4.4
     */
    @Override
    public boolean removeIf(final Predicate<? super E> filter) {
        synchronized (lock) {
            return decorated().removeIf(filter);
        }
    }

		...
}
```

디폴트 메소드에 대한 조치는?

구현한 인터페이스의 디폴트 메소드를 재정의하고, 다른 메소드에선 디폴트 메소드를 호출하기 전에 필요한 작업을 수행하도록 한다.

## 요약 정리

- **즉, 디폴트 메소드는 (컴파일에 성공해도) 기존 구현체에 런타임 오류를 일으킬 수 있다.**
- 기존 인터페이스에 디폴트 메소드를 새로 추가하는 일은 꼭 필요한 경우가 아니면 피해라.
- **하지만, 새로운 인터페이스를 만들 때는 디폴트 메소드가 표준적인 메소드를 구현하는 데 아주 유용한 수단**이며, 인터페이스를 더 쉽게 구현하여 활용할 수 있도록 돕는다.
- 새로운 인터페이스라면 릴리스 전에 반드시 테스트를 거쳐야 한다.
    - 서로 다른 방식으로 최소한 3개의 구현체는 만들어봐야 한다.
    - 각 인터페이스 인스턴스에 대해 클라이언트 코드도 여러 만들어봐야 한다.
    - **인터페이스를 릴리스 한후에 결함을 수정하는 일도 가능할 수 있지만, 그것에 기대선 안된다.**