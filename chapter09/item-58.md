# Item 58. 전통적인 for 문보다는 for-each 문을 사용하라
스트림이 제격인 작업(item 45) 이 있고 반복문이 제격인 작업이 있다.<br>
다음은 전통적인 for 문으로 컬렉션을 순회하는 코드이다.
```java
// 컬렉션 순회
for (Iterator<Element> i = c.iterator(); i.hasNext(); ) {
	Element e = i.next();
    ... //e로 무언가를 한다.
}
```
다음은 전통적인 for 문으로 배열을 순회하는 코드이다.
```java
// 배열 순회
for (int i = 0; i < a.length; i++) {
    int b = a[i]
    // 무언가를 한다.
}
```

전통적인 for 문의 문제점
- 반복자와 인덱스 변수는 코드를 지저분하게 할 뿐이다.
    - 우리에게 필요한 것은 단지 원소들뿐이다.
- 사용할 요소가 많아지면 오류가 생길 가능성이 높아진다.
    - 1번 예시에선 반복자(i)가 3번 등장하며, 2번 예시에선 인덱스 변수(i)가 4번 등장한다.
    - 따라서 변수를 잘못 사용할 가능성이 높아진다.
    - 만약, 문맥 상 잘못된 변수를 사용해도 컴파일러가 잡아주지 못할 수도 있다.
- 또한 컬렉션이냐 배열이냐에 따라 코드 형태가 달라지므로 주의해야 한다.

## for-each 문
- 위의 전통적인 for 문에서의 문제점은 for-each 문으로 모두 해결할 수 있다.
- for-each 문의 정석 이름은 '향상된 for 문'이다.
- 장점
    - 반복자와 인덱스를 사용하지 않으므로 코드 가독성도 좋아지고, 오류가 날 일도 없다.
    - 또한 하나의 관용구로 컬렉션과 배열을 일관되게 처리할 수 있어 어떤 컨테이너를 다루는 지 신경 안써도 된다.
```java
// 컬렉션 혹은 배열 순회
for (Element e : elements) {
	... //e로 무언가를 한다.
}
```
- 콜론(:)은 "안의(in)"라고 읽으면 된다.
- 이 반복문은 "elements 안의 각 원소 e에 대해"라고 읽는다.
- 반복 대상이 컬렉션이든 배열이든 for-each 문을 사용해도 속도는 그대로이다.

### 컬렉션을 중첩해 순회하는 경우
다음 전통적인 for 문으로 반복자를 사용할 때
```java
for (Iterator<Suit> i = suits.iterator(); i.hasNext(); ) {
    for (Iterator<Rank> j = ranks.iterator(); j.hasNext(); )
        deck.add(new Card(i.next(), j.next())); // i.next() 잘못 호출
}
```
- 위와 같이 하기 쉬운 실수를 저지를 수 있다. 
- 의도는 모든 suit 와 rank 쌍을 만드는 것이지만, 안쪽 반복문에서 호출되어 rank 하나에 하나씩만 매핑되고 끝난다.
- 따라서 만약 숫자(suit) 가 바닥나면 반복문에서 NoSuchElementException 을 던질 것이다.
- 하지만, 만약 운이 나쁘게 바깥 suit 컬렉션 크기가 더 컸다면?
    - 예외도 던지지 않고 종료한다. 의도한 결과도 얻을 수 없다.
```java
// 컬렉션 중첩 순회
for (Iterator<Suit> i = suits.iterator(); i.hasNext(); ) {
    Suit suit = i.next();
    for (Iterator<Rank> j = ranks.iterator(); j.hasNext(); )
        deck.add(new Card(suit, j.next()));
}
```
- 위와 같이 바깥 원소를 저장하는 변수를 하나 추가하여 바깥으로 빼면 해결할 순 있다.

위 코드를 for-each 문으로 더 간단히 해결할 수 있다.
```java
for (Suit suit : suits)
    for (Rank rank : ranks)
        deck.add(new Card(suit, rank));
```

## for-each 문을 사용할 수 없는 상황
- 파괴적인 필터링(destructive filtering):
    - 만약 for-each 에서 컬렉션 원소를 제거하려 한다면 ConcurrentModificationException 예외가 발생한다.
    - 컬렉션을 순회하면서 원소를 제거해야 한다면 반복자의 remove() 를 호출해야 한다.
    - java 8부턴 Collection 의 removeIf 메서드로 컬렉션을 명시적 순회하지 않고 제거할 수도 있다.
        - `list.removeIf(s -> s.equals("b"));`
- 변형(transforming):
    - for-each 에선 컬렉션을 순회하면서 해당 원소의 값을 변경할 순 없다.
    - 순회하면서 그 원소의 값 일부 혹은 전체를 교체해야 한다면 반복자나 배열의 인덱스를 사용해야 한다.
- 병렬 반복(parallel iteration):
    - 여러 컬렉션을 병렬로, 한번에 순회해야 한다면 각각의 반복자와 인덱스로 명시적으로 제어해야 한다.
    - 앞선, suit 와 rank 에서 반복자 순회의 잘못된 예제 코드가 그렇다.
    ```java
    for (int i = 0; i < a.size(); i++) {
        System.out.println(a.get(i) + b.get(i)); 
    }
    ```

따라서 위 상황에선 일반 for 문을 사용하되 앞서 언급한 문제점들을 주의해야 한다.

## Iterable 인터페이스
- for-each 문은 컬렉션과 배열은 물론 Iterable 인터페이스를 구현했다면 무엇이든지 순회할 수 있다.
- Iterable 인터페이스는 다음과 같이 하나의 메서드를 가진다.
```java
public interface Iterable<E> {
    Iterator<E> iterator(); // 이 객체의 원소를 순회하는 반복자를 반환한다.
}
```
- 원소들의 묶음을 표현하는 타입을 작성해야 한다면 Iterable 구현을 고민해보아라
- Collection 은 구현하지 않아도 Iterable 구현은 고려해라
- Iterable 만 구현해 놓아도 for-each 를 사용할 수 있어 좋다.

## 요약 정리
- 일반 for 문과 비교했을 때 for-each 문은 명료하고, 유연하고, 버그를 에방한다.
- 성능 저하도 없다.
- 가능한 모든 곳에서 for 문보다 for-each 문을 사용하자

