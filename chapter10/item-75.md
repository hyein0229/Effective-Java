# Item 75. 예외의 상세 메시지에 실패 관련 정보를 담으라
## 스택 추적(stack trace) 정보
- 예외를 잡지 못하여 프로그램이 실패 시엔 자바 시스템은 스택 추적 정보를 자동 출력한다.
- 스택 추적은 예외 객체의 toString() 메서드로 얻는 문자열로, 보통 예외 클래스 이름 뒤 붙는 설명이다.
- 해당 정보는 프로그래머가 얻을 수 있는 예외에 대한 유일한 정보이다.
    - 그 실패를 재현하기 어렵다면 더 자세한 정보를 얻기가 어렵거나 불가능하다.
    - 따라서 toString() 메서드에 실패 원인에 대한 정보를 최대한 많이 담아 반환하는 것이 중요하다.
- 예외 상세 메시지에 실패 순간의 상황을 정확히 담아 반환하여라
```shell
Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException: Index 5 out of bounds for length 3
	at Main.process(Main.java:13)
	at Main.start(Main.java:9)
	at Main.main(Main.java:5)
```
## 예외 상세 메시지 작성
- **실패 순간을 정확히 포착하기 위해선 예외에 관여된 모든 매개변수와 필드값을 메시지에 담아라**
    - 예를 들어 IndexOutOfBoundsException 의 경우 범위 최솟값, 최댓값, 접근 인덱스를 담아야 한다.
    - 잘못된 인덱스 범위로 접근했는지, 아니면 최솟값이 최댓값보다 커 내부 불변식이 깨졌는지 등을 알 수 있다.
    - 관련 데이터를 모두 담아야 하지만 장황할 필요는 없다.
    - 프로그래머는 문서와 소스코드를 함께 살펴볼 것이므로 거기서 얻을 수 있는 정보를 길게 담을 필욘 없다.
- 예외의 상세 메시지와 최종 사용자에게 보여줄 오류 메시지를 혼동해선 안된다.
    - 최종 사용자에겐 친절한 안내 메시지를 보여주어야 한다. (현지어로 번역해주기도 한다.)
    - 하지만 예외 상세 메시지는 가독성보단 담긴 내용이 훨씬 중요하다.
- 실패를 적절히 포착하기 위해선 필요한 정보를 예외 생성자에서 받아 상세 메시지를 미리 생성할 수도 있다.
    - 상세 메시지를 만들어내는 코드를 예외 클래스 안으로 모아줄 수 있다.
    - 따라서 사용자가 상세 메시지를 만드는 작업을 중복하지 않아도 되는 장점이 있다.
```java
/**
 * IndexOutOfBoundsException을 생성한다.
 *
 * @param lowerBound 인덱스의 최솟값
 * @param upperBound 인덱스의 최댓값 + 1
 * @param index 인덱스의 실젯값
 */
public IndexOutOfBoundsException(int lowerBound, int upperBound, int index) {
   // 실패를 포착하는 상세 메시지를 생성한다.
   super(String.format("최솟값: %d, 최댓값: %d, 인덱스: %d", lowerBound, upperBound, index));
   
   // 프로그램에서 이용할 수 있도록 실패 정보를 저장해둔다.
   this.lowerBound = lowerBound;
   this.upperBound = upperBound;
   this.index = index;
}
```
- 기존 생성자는 String 을 생성자에서 받는다. 
- java 9부턴 IndexOutOfBoundsException 에 정수 인덱스 값을 받는 생성자가 추가되었다.
- 하지만, 아쉽게도 범위 최솟값, 최댓값까지 받진 않는다.

## 예외 클래스 접근자 메서드 제공
- 예외는 실패와 관련한 정보를 얻을 수 있는 메서드를 제공하는 것이 좋다.
    - 예를 들어 위 IndexOutOfBoundsException 에선 범위 최솟값, 최댓값, 인덱스를 제공할 수 있다.
    - getIndex() 와 같은 접근자로 인덱스 정보를 가져올 수 있도록 한다.
- 물론, 검사 예외보다 비검사 예외에서는 중요도가 떨어진다.
- 이는 디버깅하는 데에 유용하다.

## 요약 정리
- 스택 추적은 예외 상황에서의 정보를 제공하여 프로그래머의 디버깅을 돕는다.
- 따라서 예외 상황을 알려주는 예외 상세 메시지는 실패 원인을 정확히 포착할 수 있도록 모든 매개변수나 필드값을 담아라
- 예외는 실패와 관련한 정보를 얻을 수 있는 접근자 메서드를 제공하는 것이 좋다.