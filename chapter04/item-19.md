# Item 19. 상속을 고려해 설계하고 문서화하라. 그러지 않았다면 상속을 금지하라

item 18 에선 상속을 염두에 두고 설계되지 않았고, 상속할 때의 문제를 문서화해놓지 않은 ‘외부’ 클래스를 상속할 때의 문제점을 알아보았다. 그렇다면, 상속을 사용하기 위한 상속을 고려한 설계와 문서화는 무엇일까

## 상속용 클래스 정의시 주의점

### 1. 상속용 클래스는 재정의할 수 있는 메소드들을 내부적으로 어떻게 이용하는지(자기사용) 문서로 남겨야 한다.

즉, 공개된 API 내부에서 클래스 자신의 또 다른 메소드를 호출할 수 있으므로 어떤 것이 호출되고, 어떤 순서로 호출되는지, 각각의 호출 결과가 이어지는 처리에 어떤 영향을 주는지를 담아야 한다.

하지만 이 방식은 “좋은 API 문서란 어떻게가 아닌 ‘무엇’을 하는지를 설명해야 한다” 라는 격언과는 대치된다. 그러나 상속이 캡슐화를 해치므로 안전한 상속을 위해선 ‘내부 구현 방식’ 설명은 어쩔 수 없다.

**Implementation Requirements**

- API 문서의 메소드 설명 끝에선 종종 “Implementation Requirements”로 시작하는 절을 볼 수 있다.
- 메소드의 내부 동작 방식을 설명하는 곳
- 메서드 주석에 @implSpec 태그를 붙이면 자바독 도구가 생성해준다.
- [AbstractionCollection (Java SE 11 & JDK 11)](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/AbstractCollection.html#remove(java.lang.Object))

> 이 메서드는 컬렉션을 순회하며 주어진 원소를 찾도록 구현되었다. 주어진 원소를 찾으면 반복자의 remove 메서드를 사용해 컬렉션에서 제거한다. 이 컬렉션이 주어진 객체를 갖고 있으나, 이 컬렉션의 iterator 메서드가 반환한 반복자가 remove 메서드를 구현하지 않았으면 UnsupportedOperationException을 던지니 주의하자.
> 

```java
public boolean remove(Object o) {
    Iterator<E> it = iterator();
    if (o==null) {
        while (it.hasNext()) {
            if (it.next()==null) {
                it.remove();
                return true;
            }
        }
    } else {
        while (it.hasNext()) {
            if (o.equals(it.next())) {
                it.remove();
                return true;
            }
        }
    }
    return false;
}
```

이를 통해 iterator 메소드를 재정의하면 remove 의 동작에 영향을 줄 수 있음을 알 수 있다. 반환받은 반복자의 동작이 remove 동작에 영향을 준다는 것도 명시하고 있다.

---

### 2. 효율적인 하위 클래스를 어려움 없이 만들기 위해선 클래스의 내부 동작 과정에 끼어들 수 있는 hook을 잘 선별하여 protected 메소드 형태로 공개해야 할 수도 있다.

즉, 하위 클래스가 필요한 부분만 고성능을 위해 재정의하여 갈아끼울 수 있도록 제공하는 hook 메소드이다. <br> 내부 설계만을 위해 존재하므로 protected 로 선언한다. 

java.util.AbsractList의 removeRange 메서드 설명 

> protected void removeRange(int fromIndex, int toIndex) fromIndex(포함)부터 toIndex(미포함)까지의 모든 원소를 이 리스트에서 제거한다. toIndex 이후의 원소들은 앞으로 (index만큼씩) 당겨진다. 이 호출로 리스트는 'toIndex - fromIndex'만큼 짧아진다. (toIndex == fromIndex라면 아무 효과도 없다.) 이 리스트 혹은 이 리스트의 부분리스트에 정의된 clear 연산이 이 메서드를 호출한다. 리스트 구현의 내부 구조를 활용하도록 이 메서드를 재정의하면 이 리스트와 부분리스트의 clear 연산 성능을 크게 개선할 수 있다. <br><br>
**Implementation Requirements**: 이 메서드는 fromIndex에서 시작하는 리스트 반복자를 얻어 모든 원소를 제거할 때까지 ListIterator.next와 ListIterator.remove를 반복 호출하도록 구현되었다. **주의: ListIterator.remove가 선형 시간이 걸리면 이 구현의 성능은 제곱에 비례한다.** <br><br>
 Parameters: fromIndex 제거할 첫 원소의 인덱스 toIndex 제거할 마지막 원소의 다음 인덱스
> 

```java
// 이 메소드를 List 하위 구현체들이 오버라이딩하여 효율적으로 제거할 수 있도록 함
protected void removeRange(int fromIndex, int toIndex) {
    ListIterator<E> it = listIterator(fromIndex);
    for (int i=0, n=toIndex-fromIndex; i<n; i++) {
        it.next();
        it.remove();
    }
}
```

- clear 메소드 내부적으로 removeRange 를 호출하여 제거
- List 구현체를 사용하는 클라이언트는 removeRange 메소드에는 관심이 없다.
- 단지 하위 클래스에서 부분리스트의 clear 메소드를 고성능으로 쉽게 만들기 위함이다.
- removeRange 메서드가 없다면?
    - remove 메소드를 한번 호출하면 하나 지우고 뒤 요소들을 모두 shift → O(n)
    - (제거할 원소 수의) 제곱에 비례해 성능이 느려지거나 메커니즘을 밑바닥부터 새로 구현해야 했을 것이다.

---

### 3. 상속용으로 설계한 클래스는 배포 전에 반드시 하위 클래스를 만들어 검증해라.

- 상속용 클래스를 시험하는 방법은 직접 하위 클래스를 만들어보는 것이 ‘유일한’ 방법이다.
- 꼭 필요한 protected 멤버를 놓쳤다면 그 빈자리가 드러난다.
- 거꾸로, 하위 클래스를 여러 개 만들때까지 쓰이지 않는 protected 멤버는 private 어야 할 가능성이 크다.
- 하위 클래스는 3개 정도, 이 중 하나 이상은 제3자가 작성해봐야 한다.
- 널리 쓰일 클래스를 상속용으로 설계한다면 반드시 문서화한 내부 사용 패턴과, protected 메서드와 필드를 구현하면서 이 결정이 그 클래스의 성능과 기능에 영원한 족쇄가 될 수 있음을 인식해야 한다.

---

### 4. 상속용 클래스의 생성자는 직접적이든 간접적이든 재정의 가능 메소드를 호출해서는 안된다.

상위 클래스의 생성자는 하위 클래스의 생성자보다 먼저 실행되므로 상의 클래스의 생성자에서 **재정의 가능 메소드를 호출한다면 해당 메소드가 하위 클래스의 생성자보다 먼저 호출하게 된다.** 이때 재정의 가능 메소드에서 생성자에서 초기화하는 값에 의존하게 된다면 오동작할 것이다.

> private, final, static 메소드는 재정의가 불가능하므로 생성자에서 호출해도 된다.
> 

```java
public class Super {
    // 잘못된 예시 - 생성자가 재정의 가능 메서드를 호출한다.
    public Super() {
        overrideMe(); // 하위 클래스에 오버라이딩된 메소드 호출
    }

    public void overriedMe() {

    }
}
```

```java
public final class Sub extends Super {
    // 초기화되지 않은 final 필드, 생성자에서 초기화한다.
    private final Instant instant;

    Sub() {
		// super();
        instant = Instant.now();
    }

    // 재정의 가능 메서드, 상위 클래스의 생성자가 호출한다.
    @Override public void overrideMe() {
        System.out.println(instant); // 초기화 필요한 값 사용 - null
    }

    public static void main(String[] args) {
        Sub sub = new Sub();
        sub.overrideMe();
    }
}
```

- 프로그램은 instant 를 두 번 출력하지 못하고 첫 번째는 null 을 출력한다.
- 상위 클래스 생성자에 의해 overrideMe 가 초기화도 전에 호출되기 때문이다.
- overrideMe 내부에서 instant 객체의 메소드를 호출한다면 NullPointerException 이 발생할 것이다.

---

### 5. clone과 readObject 모두 직접적으로든 간접적으로든 재정의 가능 메서드를 호출해서는 안 된다.


- Cloneable 과 Serializable 인터페이스는 상속용 설계의 어려움을 한층 더한다. 즉, 해당 인터페이스의 구현체를 상속용으로 만드는 것은 좋지 못하다.
- clone과 readObject 메소드는 생성자와 비슷한 효과를 낸다. (새로운 객체를 생성한다) 따라서, 이 메소드를 정의 시에도 생성자와 비슷한 제약을 갖는다는 점을 주의해라
- readObject의 경우 하위 클래스의 상태가 미처 다 역직렬화되기 전에 재정의한 메소드부터 호출한다.
- clone의 경우 하위 클래스의 clone 메소드가 복제본의 상태를 (올바른 상태로) 수정하기 전에 재정의한 메소드를 호출한다.
- clone 에서 완벽히 깊은 복사가 이루어지지 않아 복제본의 어딘가에서 원본 객체의 데이터를 참조하고 있다면 원본 객체까지 잘못될 수도 있다.

---

### 6. Serializable을 구현한 상속용 클래스가 readResolve나 write Replace 메서드를 갖는다면 이 메서드들은 private이 아닌 protected로 선언해야 한다.

- private 로 선언한다면 하위 클래스에서 무시되므로 jvm 에서 직렬화/역질렬화 시 호출이 불가능하다.
- 상속을 허용하기 위해 내부 구현을 클래스 API 로 공개하는 예 중 하나이다.

즉, 클래스를 상속용으로 설계하기 위해선 엄청난 노력이 들고 그 클래스에 생기는 제약도 커진다.

## 상속을 금지하는 법

일반적인 구체 클래스는 final 도 아니고 상속용으로 설계되거나 문서화되지 않았다. 이런 구체 클래스를 상속하면 구체 클래스에 변화가 있을 때마다 하위 클래스에도 오동작하게 만들 것이다.

**상속을 금지하도록 하는 방법은 두 가지이다.**

- final 로 클래스 선언한다.
- 생성자를 private 이나 package-private 선언하고 public 정적 팩토리를 만드는 것이다.

핵심 기능을 정의한 인터페이스가 있고, 클래스가 그 인터페이스를 구현했다면 상속을 금지해도 개발에 문제 없다. 래퍼 클래스 패턴은 기능을 확장하는데 상속 대신의 더 나은 대안이라고 할 수 있다.

___

### 그러나, 표준 인터페이스가 없는 구체 클래스가 있고 상속을 허용해야 한다면?

클래스 내부에서 재정의 가능 메소드를 사용하지 않도록 해야 한다.

즉, 재정의 가능 메소드를 호출하는 “자기사용” 코드는 완벽히 제거해라. 그렇다면 상속해도 그렇게 위험하지 않은 하위 클래스를 만들 수 있다. 메소드를 재정의해도 내부에서 사용하지 않으므로 다른 메소드의 동작에 아무런 영향을 주지 않기 때문이다.

**클래스의 동작을 유지하면서 재정의 가능 메소드를 사용하는 코드를 제거할 수 있는 방법**

- 각각의 재정의 가능 메소드는 자신의 내부 코드를 private ‘도우미 메소드’로 옮기고 이것을 호출하도록 한다.
- 재정의 가능 메소드를 호출하는 메소드에서도 대신 이 ‘도우미 메소드’를 호출하도록 한다.

```java
public class Parent {
    public void doWork() {
        log();  // 재정의 가능한 메서드를 직접 호출
    }
		
	// 재정의 가능 메소드
    protected void log() {
        System.out.println("Parent logging");
    }
}
```

```java
public class Parent {
    public void doWork() {
        logHelper();  // → 항상 내부 고정된 동작을 수행
    }

    public void log() {
        logHelper();  // → 하위 클래스가 원하면 이걸 오버라이드 가능
    }

    // private 도우미 메서드: 재정의 불가능
	  // 즉 하위에서 log 를 재정의해도 자신의 log 본문 코드를 실행하게 됨 
    private void logHelper() {
        System.out.println("Parent logging");
    }
}
```

## 요약 정리

- 상속용 클래스를 설계하기 위해선 자기사용 패턴을 모두 문서화해야 한다.
- 문서화하고 지키지 않으면 내부 구현 방식을 믿고 활용하는 하위 클래스를 오동작하게 만들 수 있다.
- 또한 효율적으로 하위 클래스를 설계하기 위해선 일부 메소드를 protected 로 제공해야 한다. : 훅(hook)
- 클래스를 확장해야 하는 명확한 이유가 없으면 상속을 금지하는 편이 나을 수 있다.
- 상속 금지를 위해 final 선언하거나 외부에서 생성자 접근을 막도록 하면 된다.