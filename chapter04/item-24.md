# Item 24. 멤버 클래스는 되도록 static 으로 만들라
## 중첩 클래스란?

- 다른 클래스 안에 정의된 클래스를 말한다.
- 자신을 감싼 바깥 클래스에서만 쓰여야 하며, 그 외의 쓰임새가 있다면 톱레벨로 만들어야 한다.
- 종류
    - 정적 멤버 클래스
    - (비정적) 멤버 클래스
    - 익명 클래스
    - 지역 클래스

**중첩 클래스를 언제 그리고 왜 사용해야 할까?**

## 정적 멤버 클래스

- 클래스 내부에 static 으로 선언된 클래스
- 바깥 클래스의 private 멤버에도 접근할 수 있다.
    - private static 이면 그냥 참조 가능
    - static 이 아니면 바깥 인스턴스를 직접 매개변수로 받아 인스턴스.필드명 접근 가능
- 다른 점은 일반 클래스와 동일하다.
- 흔히 바깥 클래스와 함께 쓰일 때만 유용한 public 도우미 클래스로 쓰인다.
- 외부 클래스 로드 여부 상관없이 자신이 참조되는 시점에 클래스 로더에 의해 로드된다.

```java
public class Calculator {
    public enum Operation { // 열거 타입도 암시적 static 이다.
        PLUS, MINUS, MULTIPLE, SUBTRACT
    }
}

// Calculator.Operation.PLUS 로 참조
```

## 비정적 멤버 클래스

- 정적 멤버 클래스에서 구문상 static 만 빠진 것이지만, 의미 차이가 크다.
- 비정적 멤버 클래스의 인스턴스는 바깥 클래스 인스턴스와 암묵적으로 연결된다.
- 따라서, 비정적 멤버 클래스의 인스턴스 메소드에서 정규화된 this (클래스명.this) 를 사용해 바깥 인스턴스의 메소드를 호출하거나 참조를 가져올 수 있다.
- 만약, **중첩 클래스의 인스턴스가 바깥 인스턴스와 독립적으로 존재할 수 있다면 정적 멤버로 만들어야 한다.**
    - 비정적 멤버 클래스는 바깥 인스턴스 없이는 생성할 수 없기 때문이다.
- 비정적 멤버 클래스의 인스턴스와 바깥 인스턴스 사이의 관계는 멤버 클래스가 인스턴스화될 때 확립되며, 더 이상 변경할 수 없다. 이 관계는 보통 멤버 클래스의 생성자를 호출할 때 만들어진다.
- **어댑터를 정의할 때 자주 쓰인다.**
    - 어떤 클래스의 인스턴스를 감싸 다른 클래스의 인스턴스처럼 보이게 하는 뷰로 사용
    - Map 인터페이스의 구현체들은 보통 자신의 컬렉션 뷰 (keySet 등) 구현 시 비정적 멤버 클래스 사용
    - Set 과 List 와 같은 다른 컬렉션 인터페이스도 자신의 반복자를 구현할 때 사용
            
    
    ```java
    // 자신의 반복자 구현
    public class MySet<E> extends AbstractSet<E> {
     
      @Override 
      public Iterator<E> iterator(){
        return new MyIterator();
      }
    
      private class MyIterator implements Iterator<E> {
        ...
      }
    }
    ```
    
    ```java
    public class HashMap<K,V> extends AbstractMap<K,V>
        implements Map<K,V>, Cloneable, Serializable {
        ...
        public Set<K> keySet() {
            Set<K> ks = keySet;
            if (ks == null) {
                ks = new KeySet(); 
                keySet = ks;
            }
            return ks;
        }
    	  
        // 어댑터
        // map 인스턴스를 감싸 set 으로 변환
        // 자신의 컬렉션 뷰를 구현할 때 비정적 멤버 클래스를 사용
        final class KeySet extends AbstractSet<K> {
            public final int size()                 { return size; }
            public final void clear()               { HashMap.this.clear(); }
            public final Iterator<K> iterator()     { return new KeyIterator(); }
            public final boolean contains(Object o) { return containsKey(o); }
            public final boolean remove(Object key) {
                return removeNode(hash(key), key, null, false, true) != null;
            }
            public final Spliterator<K> spliterator() {
                return new KeySpliterator<>(HashMap.this, 0, -1, 0, 0);
            }
            public final void forEach(Consumer<? super K> action) {
                Node<K,V>[] tab;
                if (action == null)
                    throw new NullPointerException();
                if (size > 0 && (tab = table) != null) {
                    int mc = modCount;
                    for (Node<K,V> e : tab) {
                        for (; e != null; e = e.next)
                            action.accept(e.key);
                    }
                    if (modCount != mc)
                        throw new ConcurrentModificationException();
                }
            }
        }
    }
    ```
    

---

### 바깥 인스턴스에 접근할 일이 없다면 무조건 static 붙여서 정적 멤버 클래스로 만들자.
- **static 생략하여 비정적으로 만들 시 바깥 인스턴스로의 숨은 외부 참조를 가지게 된다.**
    - 참조를 저장하려면 시간과 공간이 소비된다.
    - 더 심각한 문제는 GC 가 바깥 클래스의 인스턴스를 수거하지 못하여 메모리 누수 문제가 발생할 수 있다.
    - 참조가 눈에 보이지 않으므로 때때로 심각한 상황을 초래할 수 있다.

### private 정적 멤버 클래스는 바깥 클래스가 표현하는 객체의 구성요소를 나타낼 때 쓴다.

- 예시로 Map 구현체는 키-값 쌍을 표현하는 엔트리 객체들을 가지고 있다.
- 하지만, 엔트리는 맵과 연관은 있지만 엔트리의 메소드들은 맵을 직접 사용하진 않는다.
- 따라서, 비정적 멤버 클래스로 사용하는 것은 낭비고, private 정적 멤버가 가장 알맞다.

멤버 클래스가 공개된 클래스의 public 이나 protected 멤버라면 정적이냐 아니냐는 두 배로 중요해진다. 멤버 클래스 역시 공개 API 가 되므로, 혹시라도 향후 릴리스에서 static 을 붙이면 하휘 호환성이 깨진다. 즉, 클라이언트 코드에서 오류가 발생한다.

## 익명 클래스

- 바깥 클래스의 멤버도 아니다.
- **멤버와 달리, 쓰이는 시점에 선언과 동시에 인스턴스가 만들어진다.**
- 코드의 어디서든 만들 수 있다.
- 오직 비정적인 문맥에서 사용될 때만 바깥 클래스의 인스턴스를 참조할 수 있다.
- 정적 문맥(static)에서라도 상수 변수 이외의 정적 멤버는 가질 수 없다.
- **람다를 지원하기 전엔 즉석에서 작은 함수 객체나 처리 객체를 만드는데 익명 클래스를 사용했다. 이제는 람다에게 자리를 물려줬다.**
- 또 다른 주 쓰임은 정적 팩토리 메소드를 구현할 때이다.

람다 지원 전 익명 클래스로 Comparator 구현한 예시

```java
List<Integer> list = Arrays.asList(10, 5, 6, 7, 1, 3, 4);
Collections.sort(list, new Comparator<Integer>() {
    @Override
    public int compare(Integer o1, Integer o2) {
        return Integer.compare(o1, o2);
    }
});
System.out.println(list);
```

람다를 사용한 예시

```java
Collections.sort(list, (o1, o2) -> Integer.compare(o1, o2));

Collections.sort(list, Comparator.comparingInt(o -> o));
```

## 지역 클래스

- 가장 드물게 사용된다.
- 지역변수를 선언할 수 있는 곳이면 어디서든 선언 가능하고, 유효 범위도 지역 변수와 같다.
- 멤버 클래스처럼 이름이 있고 반복해서 사용할 수 있다.
- 익명 클래스처럼 비정적 문맥에서 사용될 때만 바깥 인스턴스를 참조할 수 있다.
- 정적 멤버는 가질 수 없고, 가독성을 위해 짧게 작성해야 한다.

```java
public static void main(String[] args) {
    class MyClass{ 
      
    }
    MyClass myClass = new MyClass();
}
```

## 요약 정리

- 메소드 밖에서 사용해야 하거나 메소드 안에 정의하기엔 너무 길면 멤버 클래스로 만든다.
- ***멤버 클래스의 인스턴스가 바깥 인스턴스를 참조한다면 비정적으로, 아니면 static 으로 만들자.***
- 중첩 클래스가 한 메서드 안에서만 쓰이면서 그 인스턴스를 생성하는 지점이 단 한 곳이고 해당 타입으로 쓰기에 적합한 클래스나 인터페이스가 이미 있다면 익명 클래스로 만들고, 그렇지 않다면 지역 클래스로 만들자.