# Item 18. 상속보다는 컴포지션을 사용하라

## 상속이 잘못된 경우?

상속이 항상 최선은 아니다. 잘못 사용하면 오류를 내기 쉬운 소프트웨어를 만들게 된다.<br>
이때의 상속은 클래스가 다른 클래스를 확장하는 (extends) 구현 상속을 말한다. 

**우리가 상속을 사용하는 이유**

- 코드를 재사용하여 중복을 없앤다.
- 변화에 대해 유연하고 확장성이 증가한다.
- 개발 시간을 단축시킨다.

**상속이 올바른 경우?**

- 모두 같은 프로그래머가 통제하는 패키지 안에서 상위, 하위 클래스를 작성
- 문서화도 잘 된 클래스
- 확장 목적으로 설계된 클래스
- is-a 관계가 확실한 경우 (즉, B가 A라고 할 수 있다.)

위와 같은 조건들에 부합한다면 상속은 문제가 없다.

그러나, 위 조건을 생각하지 못하고 ***상속을 한다면 캡슐화를 깨트릴 수 있다.***


**캡슐화?**

: 변수나 함수를 하나의 클래스로 묶고 외부에서 접근하지 못하도록 숨기는 것

---

### 상속은 캡슐화를 깨뜨린다.

- 상위 클래스가 어떻게 구현되느냐에 따라 하위 클래스의 동작에 이상이 생길 수 있다.
- 상위 클래스가 릴리스마다 구현이 달라질 경우, 건드리지 않은 하위 클래스가 오동작할 수 있다.
- 즉, 하위 클래스가 상위 클래스에 **강하게 결합 및 의존하게 되는 경우** 유연한 변화가 어렵다.

**잘못된 상속의 예시1 - 상위 클래스의 필드 타입 변경**

로또번호를 가지는 Lotto 클래스

```java
public class Lotto {
	// 로또 번호 리스트를 가짐
    protected List<Integer> lottoNumbers;

    public Lotto(List<Integer> lottoNumbers) {
        this.lottoNumbers = new ArrayList<>(lottoNumbers);
    }
		
	// 해당 숫자가 포함되어 있는지 여부를 반환
    public  boolean contains(Integer integer) {
        return this.lottoNumbers.contains(integer);
    }
    ...
}
```

Lotto 클래스를 상속하고 있는, 당첨로또번호를 가지는 WinningLotto 클래스

```java
public class WinningLotto extends Lotto {
    private final BonusBall bonusBall;

    public WinningLotto(List<Integer> lottoNumbers, BonusBall bonusBall) {
        super(lottoNumbers);
        this.bonusBall = bonusBall;
    }
		
	// 입력받은 로또번호와 당첨번호가 몇 개 일치한지 카운트 반환
    public long compare(Lotto lotto) {
        return lottoNumbers.stream()
            .filter(lotto::contains)
            .count();
    }
    ...
}
```

만약 이때, Lotto 클래스의 List<Integer> lottoNumbers 가 int[] 배열로 바뀐다면?

```java
public class Lotto {
    protected int[] lottoNumbers; // int[] 로 타입 변경
		
		// int[] 로 타입 변경
    public Lotto(int[] lottoNumbers) {
        this.lottoNumbers = lottoNumbers;
    }
		
    public boolean contains(Integer integer) {
        return Arrays.stream(lottoNumbers)
            .anyMatch(lottoNumber -> Objects.equals(lottoNumber, integer));
    }
    ...
}
```

```java
public class WinningLotto extends Lotto {
    private final BonusBall bonusBall;

    // 오류가 발생한다.
    public WinningLotto(List<Integer> lottoNumbers, BonusBall bonusBall) {
        super(lottoNumbers); // int[] 타입을 넘겨야 함 - 오류
        this.bonusBall = bonusBall;
    }

    // 오류가 발생한다.
    public long compare(Lotto lotto) {
        return lottoNumbers.stream() // int[] 이므로 list.stream 불가 - 오류
            .filter(lotto::contains)
            .count();
    }
}
```

부모의 로또번호 변수에 강한 의존을 맺고 있는 하위 클래스 WinningLotto 는 오류가 발생한다. 따라서 이를 해결하기 위해선 상위 클래스 변화에 맞춰 하위 클래스의 코드를 일일이 수정해주어야 한다.

**잘못된 상속의 예시2 - 메소드 재정의**

```java
/*
* HashSet 의 요소를 몇번 삽입했는지 갯수를 출력하기 위해만든 클래스
* */
public class CustomHashSet<E> extends HashSet {
    private int addCount = 0;

    public CustomHashSet(){}

    public CustomHashSet(int initCap, float loadFactor){
        super(initCap,loadFactor);
    }
		
		/*
			super.addAll 내부에서 재정의한 메소드가 호출됨
			java 는 런타임의 실제 객체를 기준으로 메소드 호출하므로 (다형성)
		*/
    @Override
    public boolean add(Object o) {
        addCount++; // 여기서 호출될 때마다 1씩 더해짐 
        return super.add(o);
    }

    @Override
    public boolean addAll(Collection c) {
        addCount+=c.size(); // 여기서 3 더해지고 
        return super.addAll(c); // 여기서 또 3 더해짐
    }

    public int getAddCount() {
        return addCount;
    }

}
```

```java
List<String> test = Arrays.asList("아","이","스");
customHashSet.addAll(test);
```

- addCount 값이 3개일 것을 기대했지만 6으로 출력됨
- super.addAll() 내부에서 각 원소를 add 메소드를 호출하여 더함 → **하지만, add 가 재정의되어 있음**
- 재정의한 add 메소드가 호출되면서 addCount++ 에 의해 하나씩 또 더해짐

HashSet 의 addAll 

```java
public boolean addAll(Collection<? extends E> c) {
    boolean modified = false;
    for (E e : c)
        if (add(e)) // "자기사용(self-use)"
            modified = true;
    return modified;
}
```

이를 해결하기 위해선 overriding 한 add 나 addAll 메소드 자체를 없애면 된다.

그러나, 이건 상속받고 있는 HashSet 클래스가 추후 변경되면 치명적인 오류를 줄 수 있다. 

**오버라이드를 하지 않고 새로운 메소드를 추가한다면?**

- 추후 상위에서 하위에 새롭게 만들었던 메서드와 시그니처가 같고 반환 타입만 다른 메서드를 추가되는 경우 하위 클래스에선 컴파일 에러가 나게 된다.
- 만약 시그니처와 반환타입까지 똑같다면, 의도치 않게 오버라이드를 한 것이 된다.
    - 또다시, 오버라이드로 인한 문제가 발생
- 하위에서 메소드를 작성할 땐 상위의 메소드는 존재하지 않았을테니, 상위 클래스의 메소드가 요구하는 규약을 만족하지 못할 가능성이 커진다.

## 컴포지션(composition; 구성)

- 위와 같은 상속의 단점이 존재하므로 기존의 클래스를 확장하는 것 대신,
- **새로운 클래스를 만들고 private 필드로 기존 클래스의 인스턴스를 참조하도록 하는 컴포지션 방식이 있다.**
- 기존 클래스 인스턴스가 구성요소로 쓰인다는 점에서 컴포지션이라고 부른다.

새 클래스의 인스턴스 메소드들은 private 필드로 참조하고 있는 기존 클래스의 인스턴스의 메소드를 호출하여 그 결과를 반환한다. 이 방식을 **전달(forwarding)** 이라고 한다. 즉, 새로운 클래스는 기존 클래스의 내부 구현 방식의 영향에서 벗어나고, 기존 클래스에 새 메소드가 추가되어도 영향을 받지 않는다.

ForwardingSet - 전달 클래스

```java
/*
	Set 인터페이스를 구현한 전달 클래스 생성
*/
public class ForwardingSet<E> implements Set {

    private final Set<E> set; // 기존 Set 클래스 인스턴스를 구성요소로 사용

    public ForwardingSet(Set<E> set){
        this.set=set;
    }

    @Override
    public int size() {
        return set.size(); // 기존 클래스의 인스턴스의 메소드를 호출하여 결과 반환
    }

    @Override
    public boolean isEmpty() {
        return set.isEmpty();
    }

    @Override
    public boolean contains(Object o) {
        return set.contains(o);
    }

    @Override
    public Iterator iterator() {
        return set.iterator();
    }

    @Override
    public Object[] toArray() {
        return set.toArray();
    }

    @Override
    public boolean add(Object o) {
        return set.add((E) o);
    }

    @Override
    public boolean remove(Object o) {
        return set.remove(o);
    }

    @Override
    public boolean addAll(Collection c) {
        return set.addAll(c);
    }

    @Override
    public void clear() {
        set.clear();
    }

    @Override
    public boolean removeAll(Collection c) {
        return set.removeAll(c);
    }
    ...
}
```

`래퍼 클래스` - 상속 대신 컴포지션을 사용

```java
/*
* 전달 클래스를 상속
HashSet 의 전체 요소의 갯수를 출력하기 위해만든 클래스
* */
public class CustomHashSet<E> extends ForwardingSet {
    private int addCount = 0;
		
	// Set 객체를 받아 생성 
    public CustomHashSet(Set<E> set){
        super(set);
    }

    @Override
    public boolean add(Object o) {
        addCount++;
        return super.add(o);
    }

    @Override
    public boolean addAll(Collection c) {
        addCount+=c.size();
        return super.addAll(c);
    }

    public int getAddCount() {
        return addCount;
    }

}
```

- Set 인터페이스를 구현했고, Set의 인스턴스를 인수로 받는 생성자를 제공한다.
- 임의의 Set에 계측 기능을 덧씌워 새로운 Set 으로 만든 것이 이 클래스의 핵심이다.
- 상속 방식은 구체 클래스 각각을 따로 확장해야 하고, 지원하고 싶은 상위 클래스의 생성자에 대응하는 생성자를 별도로 각각 생성해주어야 한다.

```java
CustomHashSet<String> customHashSet = new CustomHashSet<>(new HashSet<>());
List<String> test = Arrays.asList("아","이","스");
customHashSet.addAll(test);
```

CustomHashSet 은 다른 Set 인스턴스를 받아 감싸고 있는 구조로 `래퍼 클래스`라고 부르며, 다른 Set 에 요소 개수를 세는 계측 기능을 덧씌운다는 뜻에서 **데코레이터 패턴(decorator pattern)** 이라고 한다. 


>**`데코레이터 패턴 (Decorator pattern)?`**<br>
대상 객체에 대한 기능 확장이나 변경이 필요할 때 객체의 결합을 통해 서브클래싱(상속) 하는 대신 쓸 수 있는 유연한 구조의 패턴이다. 즉, 기존 객체에 새로운 기능을 추가할 수 있도록 하는 디자인 패턴이며 상속 없이 기능 확장을 한다.
데코레이터 패턴을 이용하면 필요한 추가 기능의 조합을 런타임에 동적으로 생성할 수 있다. 데코레이터할 대상 객체를 새로운 기능을 제공하는 특수 장식자 객체에 넣어 행동을 연결시켜 >새로운 기능이 추가된 행동을 하는 객체를 생성할 수 있게 된다.

로또도 컴포지션으로 바꿀 수 있다.

```java
// 1. 메소드 호출 방식으로 동작하여 캡슐화가 깨지지 않는다.
// 2. 기존 클래스 Lotto 클래스가 변경되어도 영향이 거의 없다.
public class WinningLotto {
    private Lotto lotto; // Lotto 인스턴스를 구성요소로 가짐
    private BonusBall bonusBall;
}
```

래퍼 클래스는 단점이 거의 없지만 한가지 주의해야 한다.

**래퍼 클래스는 콜백(callback) 프레임워크과는 어울리지 않는다.** 콜백 프레임워크는 자기 자신의 참조를 다른 객체에 넘겨서 다음 호출 때 사용하도록 하는데, 내부 객체는 자신을 감싸는 래퍼 객체의 존재를 모르므로 이때 자기 자신의 참조를 넘겨 다음 호출 시 래퍼가 아닌 내부 객체를 호출하게 된다. 이를 SELF 문제라고 한다. 

## 요약 정리

- 상속은 캡슐화를 깨트리고 변화에 유연하지 못하게 한다.
- 컴포지션 대신 상속을 사용하려면 다음을 고려해라
    - 확장하려는 기존 클래스의 API 에 결함이 없는가
    - 순수한 is-a 관계인가
    - 상위 클래스가 확장을 고려해 설계되어 있는가
- 상속 대신 composition & forwarding 방식을 이용하자
- 래퍼 클래스로 구현할 적당한 인터페이스가 있다면 더욱 composition 방식을 사용해라