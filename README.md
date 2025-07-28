# Effective-Java
이펙티브 자바 3판 책 스터디를 진행하며 내용 정리한 레포지토리입니다.


## 스터디 진행
1. 주 1회 오프라인 모임 
2. 해당 주차의 분량을 모두 읽고 item 별 내용 정리
3. 모여서 정리한 내용 발표, 질의


## 내용 정리
### 2장 객체 생성과 파괴
- [Item 01. 생성자 대신 정적 팩토리 메서드를 고려하라](/chapter02/item-01.md)
- [Item 02. 생성자에 매개변수가 많다면 빌더를 고려하라](/chapter02/item-02.md)
- [Item 03. 생성자나 열거 타입으로 싱글턴임을 보증하라](/chapter02/item-03.md)
- [Item 04. 인스턴스화를 막으려거든 private 생성자를 사용하라](/chapter02/item-04.md)
- [Item 05. 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라](/chapter02/item-05.md)

### 4장 클래스와 인터페이스
- [Item 15. 클래스와 멤버의 접근 권한을 최소화하라](/chapter04/item-15.md)
- [Item 16. public 클래스에선 public 필드가 아닌 접근자 메소드를 사용하라](/chapter04/item-16.md)
- [Item 17. 변경 가능성을 최소화하라](/chapter04/item-17.md)
- [Item 18. 상속보다는 컴포지션을 사용하라](/chapter04/item-18.md)
- [Item 19. 상속을 고려해 설계하고 문서화하라. 그러지 않았다면 상속을 금지하라](/chapter04/item-19.md)

### 9장 일반적인 프로그래밍 원칙
- [Item 58. 전통적인 for 문보다는 for-each 문을 사용하라](/chapter09/item-58.md)
- [Item 65. 리플렉션보다는 인터페이스를 사용하라](/chapter09/item-65.md)

### 10장 예외
- [Item 73. 추상화 수준에 맞는 예외를 던지라](/chapter10/item-73.md)
- [Item 74. 메서드가 던지는 모든 예외를 문서화하라](/chapter10/item-74.md)
- [Item 75. 예외의 상세 메시지에 실패 관련 정보를 담으라](/chapter10/item-75.md)
- [Item 76. 가능한 한 실패 원자적으로 만들라](/chapter10/item-76.md)
- [Item 77. 예외를 무시하지 말라](/chapter10/item-77.md)

### 11장 동시성
- [Item 78. 공유 중인 가변 데이터는 동기화해 사용하라](/chapter11/item-78.md)
- [Item 79. 과도한 동기화는 피하라](/chapter11/item-79.md)
- [Item 80. 스레드보다는 실행자, 태스크, 스트림을 애용하라](/chapter11/item-80.md)
- [Item 81. wait 와 notify 보다는 동시성 유틸리티를 애용하라](/chapter11/item-81.md)
- [Item 82. 스레드 안전성 수준을 문서화하라](/chapter11/item-82.md)
- [Item 83. 지연 초기화는 신중히 사용하라](/chapter11/item-83.md)
- [Item 84. 프로그램의 동작을 스레드 스케줄러에 기대지 말라](/chapter11/item-84.md)