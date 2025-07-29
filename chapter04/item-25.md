# Item 25. 톱레벨 클래스는 한 파일에 하나만 담으라

***소스 파일 하나에는 반드시 톱레벨 클래스를 하나만 담아라.***

사실 소스 파일 하나에 톱레벨 클래스를 여러 개 선언해도 자바 컴파일러는 불평하지 않는다.

하지만 아무런 득이 없고 심각한 위험을 감수해야 하는 행위이다. 하지마라.

ex. Utensil.java 파일에 두 클래스를 같이 정의

```java
class Utensil {
    static final String NAME = "pan";
}

class Dessert {
    static final String NAME = "cake";
}
```

```java
public class Main {
    public static void main(String[] args) {
		// Utensil.class 없으면 Utensil.java 컴파일하여 .class 생성
        System.out.println(Utensil.NAME + Dessert.NAME);
    }
}
```

만약, Dessert.java 파일에도 Utensil.java 파일과 같이 두 클래스를 중복 정의했다면 “중복 정의” 로 인한 컴파일 오류가 나거나 먼저 어떤 소스 파일을 컴파일하냐에 따라 동작이 달라지므로 문제가 발생한다.

```java
javac Main.java Dessert.java // 컴파일 오류 발생
javac Main.java // pancake 출력, 먼저 나온 Utensil.java 컴파일하여 생성된 .class 사용
javac Main.java Utensil.java // pancake 출력, 위와 같음
javac Dessert.java Main.java // potpie 출력, Dessert.java 컴파일하여 생성된 .class 사용
```

- Main.java 를 먼저 컴파일하면 Utensil.java 를 컴파일하게 되고 Utensil.class, Dessert.class 생성
- 이미 .class 파일이 존재하면 Main.java 컴파일 시 그것을 사용

**해결책은?**

- 단순히 톱레벨 클래스들은 서로 다른 소스 파일로 분리하면 된다.
- 굳이 여러 톱레벨 클래스를 하나의 소스 파일에 담고 싶다면 정적 멤버 클래스를 사용한다.
- 다른 클래스에 딸린 부차적인 클래스라면 정적 멤버 클래스로 만드는 쪽이 나을 것이다.
    - 읽기 좋고, private 선언하면 접근 범위도 최소로 관리할 수 있기 때문이다.

톱레벨 클래스들을 정적 멤버 클래스로 바꾼 예제

```java
public class Test {
    public static void main(String[] args) {
        System.out.println(Utensil.NAME + Dessert.NAME);
    }
    
    // private 정적 멤버 클래스
    private static class Utensil{
        static final String NAME = "pan";
    }
    
    private static class Dessert{
        static final String NAME = "cake";
    }
}
```