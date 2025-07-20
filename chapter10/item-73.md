# Item 73. 추상화 수준에 맞는 예외를 던지라

## 예외 처리에서 발생할 수 있는 문제
- 메서드가 저수준 예외를 처리하지 않고 전파해버리면 전혀 관련 없어보이는 예외를 맞닥뜨릴수 있다.
- 이러한 예외는 내부 구현 방식을 그대로 드러내어 상위 계층 API를 오염시킬 수도 있다.
- 만약 다음 릴리스에서 구현 방식을 바꿀 경우 또 다른 예외가 튀어나오게 되어 클라이언트에 영향을 끼칠 수 있다.

## 예외 번역(exception translation)
위와 같은 문제를 피하기 위해선 **상위 계층에서 저수준 예외를 잡아 자신의 추상화 수준에 맞는 예외로 바꾸어 던지는 예외 번역(exception translation)** 을 수행하는 것이 좋다.
```java
try { 
	...
} catch (LowerLevelException e) { // 저수준 예외를 잡아서
	throw new HigherLevelException(...); // 자신의 계층 수준에 맞는 예외로 변환
}
```

<br>

***`AbstractSequentialList`*** 에서 수행하는 예외 번역 예시
```java
/**
  이 리스트 안의 지정한 위치의 원소를 반환한다.
  @throws IndexOutOfBoundsException index가 범위 밖이라면, 
  즉 {@code index < 0 || index >= size()}이면 발생한다.
**/
public E get(int idex) {
   ListIterator<E> i = listIterator(index);
   try {
      return i.next();
   } catch (NoSuchElementException e) { // 예외를 잡아서 IndexOutOfBoundsException 으로 변환
   	  throw new IndexOutOfBoundsException("인덱스: " + index); 
   }
}
```
- iterator.next() 에선 원소가 없는 경우 NoSushElementException 을 던진다.
- 이때 AbstractSequentialList 의 get() 메서드에선 이 예외를 잡아 IndexOutOfBoundsException 로 번역하여 던진다.

## 예외 연쇄(exception chaining)
예외 번역 시 저수준의 예외가 디버깅에 도움이 된다면 예외 연쇄(exception chaining)을 사용하는 것이 좋다. 예외 연쇄란 **근본 원인(cause)인 저수준 예외를 고수준 예외에 실어 보내는 방식** 이다.
```java
try { 
	...
} catch (LowerLevelException cause) {
    // 저수준 예외를 고수준 예외에 실어 보낸다.
	throw new HigherLevelException(cause);
}
```
고수준 예외의 생정자는 예외 연쇄용으로 설계된 상위 클래스의 생성자에 원인(cause)을 건네준다.
최종적으로 Throwable(Throwable) 생성자까지 건네지게 한다.

```java
// 예외 연쇄용 생성자
class HigherLevelException extends Exception {
  	HigherLevelException(Throwable cause) { // 저수준 예외(원인)을 받음
  	   super(cause);
    }
}
```
- 별도의 접근자 메서드(Throwable 의 getCause())를 통해 저수준 예외를 꺼내어 볼 수 있다.
- 예외 연쇄는 문제의 원인과 고수준 예외의 스택 추적 정보를 잘 통합해주어 디버깅에 도움을 줄 수 있다.
- 대부분의 표준 예외는 예외 연쇄용 생성자를 가지고 있다. 
- 그렇지 않은 예외도 Throwable 의 initCause 메서드를 이용해 '원인'을 못박아 예외 연쇄를 구현할 수 있다.


## 예외 처리 주의사항
- **무턱대고 예외를 전파하는 방식보단 예외 번역이 우수하지만 남용해선 안된다.**
    - 가능하다면 저수준 메서드가 반드시 성공하도록 하거나, 예외가 발생하지 않도록 하는 것이 최선이다.
    - 상위 계층 메서드의 매개변수 값을 아래 계층으로 건네기 전 미리 검사하는 방법으로 예외를 방지할 수 있다. 
- 아래 계층에서 예외를 피할 수 없다면, 상위 계층에서 예외를 처리하여 API 사용자에게 전파하지 않는 것이 좋다.
    - 이 경우 발생한 예외는 java.util.logging 와 같은 적절한 로깅 기능으로 예외를 기록하면 좋다.
    - 로깅으로 예외를 전파하지 않으면서 프로그래머가 로그를 분석하여 추가 조치할 수 있도록 한다.

## 요약 정리
- 아래 계층의 예외를 예방하거나 처리할 수 없고, 그 예외를 상위 계층에 노출하기 곤란하다면 **예외 번역** 을 사용하여라
- 이때 **예외 연쇄** 를 이용하면 상위 계층에선 수준에 맞는 예외를 던지도록 하면서 근본 원인도 함께 전달할 수 있어 디버깅에 좋다.