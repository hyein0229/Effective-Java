# Item 09. try-finally 보단 try-with-resources 를 사용하라

자바 라이브러리에는 InputStream, OutputStream, java.sql.Connection 등과 같이 `close 메소드`를 호출하여 직접 닫아줘야 하는 자원이 많다. 이러한 자원들은 한정된 OS 자원들을 사용하므로 닫아주지 않으면 리소스가 낭비되는 문제가 발생한다. 자원 닫기는 놓치기 쉬워 예측할 수 없는 성능 문제로 이어진다. 안전망으로 finalizer 를 활용하고는 있지만 믿음직하지 못하다. 

전통적으로 자원이 제대로 닫힘을 보장하는 수단으로는 try-finally 가 쓰였다. 

## try-finally

```java
public static String firstLineOfFile(String path) throw IOException {
    BufferedReader br = new BufferedReader(new FileReader(path));
    try {
        return br.readLine(); // 예외 발생! -> 추적 불가
    } finally {
        br.close(); // 예외 발생! -> close 예외만 출력
    }
}
```

간단해보이지만 자원이 둘 이상이라면 코드가 너무 지저분해진다.

```java
static void copy(String src, String dst) throws IOException {
	InputStream in = new FileInputStream(src);
	try {
		OutputStream out = new FileOutputStream(dst);
		try { // 중첩된 구조 -> 가독성 떨어짐
				byte[] buf = new byte[BUFFER_SIZE];
				int n;
				while ((n = in.read(buf)) >= 0)
					out.write(buf, 0, n); 
		} finally {
				out.close(); 
		}
	} finally {
		in.close();
	}
}
```

앞의 두 코드 예제에는 모두 결함이 있다.

예외는 try 블록 br.readline() 부분과 finally 블록의 close() 에서 모두 발생 가능하나 **두 번째 예외가 첫 번째 예외를 완전히 집어 삼겨 (두 번째는 무시됨) 스택 추적 내역에 첫 번째 예외에 대한 정보는 남지 않게 될 것이다.** 이는 시스템의 디버깅을 힘들게 한다.

## try-with-resources

이 구조는 try-finally 문제를 해결하지만 해당 자원이 AutoCloseable 인터페이스를 구현해야 한다. 

AutoCloseable 은 단순히 void 를 반환하는 close 메소드 하나만 정의한 인터페이스이다. 

**닫아야 하는 자원을 뜻하는 클래스를 작성한다면 AutoCloseable 을 반드시 구현하여라**

자원을 회수하는 최선책 try-with-resources 구문

```java
public static String firstLineOfFile(String path) throw IOException {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine(); // close 에서 동시 예외 발생 시 readLine 예외가 우선시됨
    }
}
```

try-with-resources 에서는 try 블록이 끝나면 br.close() 가 자동 호출될 것이다. **만약, readLine 과 close 모두에서 예외가 발생한다면 close 에서 발생한 예외는 숨겨지고 readLine 에서 발생한 예외가 기록된다.** 이처럼 실전에선 예외 하나만 보존되고 여러 다른 예외들은 숨겨질 수도 있다. 

***숨겨진 예외들은 스택 추적 내역에 ”숨겨졌다(suppressed)” 꼬리표를 달고 출력된다.** 즉, 복수 예외들을 추적할 수 있다.* 또한 자바7에서 `Throwable`에 추가된 `getSuppressed` 메서드를 쓰면 프로그램 코드에서 가져올 수 있다.

try-with-resources + catch 문

```java
public static String firstLineOfFile(String path) throw IOException {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    } catch (Exception e) { // 예외를 잡아 처리
        return defaultVal; // 파일을 열거나 읽지 못했을 때 기본값을 반환하는 코드
    }
}
```

try-catch-finally 처럼 try-with-resources 에서도 catch 절을 같이 사용 가능하다. catch 절을 이용해서 try 문을 더 중첩하지 않고 다수의 예외를 처리할 수 있게 된다.

기존의 try-finally 에서 중첩으로 인해 지저분한 코드를 아래와 같이 개선할 수 있다.

```java
static void copy(String src, String dst) throws IOException {
	try (InputStream in = new FileInputStream(src);
			OutputStream out = new FileOutputStream(dst)) {
		byte[] buf = new byte[BUFFER_SIZE];
		int n;
		while ((n = in.read(buf)) >= 0)
			out.write(buf, 0, n);
	}
}
```

코드를 더 짧고 가독성 좋게 구현할 수 있으며 복수 예외 추적으로 문제를 진단하기에도 훨씬 좋다. 

선언된 순서대로 리소스를 열고, try 블록이 끝나면 선언 역순으로 리소스를 닫는다. → 알아서 관리해주므로 안전하다. **역순으로 닫는 이유는, 예외 발생 시 리소스 누수나 의존 관계 꼬임 등을 방지하기 위함이다**

## 요약 정리

- 자원을 회수하는 코드를 구현할 땐 try-finally 보단 try-with-resources 를 사용해라
- **코드는 더 짧고 가독성**이 좋아지고, 만들어지는 **예외 정보도 훨씬 유용**하다.
    - 예제에서 readLine 을 close 보다 더 주 예외로 판단하고 보여준다.
    - 주 예외 외의 숨겨진 예외도 스택 추적 내용을 통해 찾아볼 수 있다.
- try-finally 에서 코드가 지저분해지는 경우도 정확하고 쉽게 작성할 수 있다.
- try-with-resources 사용 시에는 닫아야 하는 자원 클래스가 AutoCloseable 을 구현해야 한다.