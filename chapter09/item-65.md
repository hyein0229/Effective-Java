# Item 65. 리플렉션보다는 인터페이스를 사용하라
## 리플렉션(Reflection)이란?
- java.lang.reflect 패키지 내 기능을 사용하여 런타임에 프로그램에서 임의의 클래스에 접근할 수 있다.
- 기능
    - Class 객체가 주어지면 그 클래스의 Constructor, Method, Field 인스턴스를 가져올 수 있다. 
    - 해당 인스턴스들로 그 클래스의 멤버 이름, 필드 타입, 메서드 시그니처 등을 가져올 수 있다.
    - Constructor, Method, Field 인스턴스를 이용하여 각각에 연결된 실제 생성자, 메서드, 필드를 조작할 수 있다.
    - 즉, 리플렉션으로 클래스의 인스턴스를 생성하거나, 메서드를 호출하거나, 필드를 조작할 수 있다는 뜻이다.
    - Method.invoke 는 어떤 클래스 객체 인스턴스의 어떤 메서드라도 호출할 수 있다.
```java
public class ReflectionExample {
    public static void main(String[] args) throws Exception {
        // 1. 클래스명으로 클래스 객체 가져온다.
        Class<?> clazz = Class.forName("MyService");

        // 2. 인스턴스를 생성한다 
        Object instance = clazz.getDeclaredConstructor().newInstance();

        // 3. sayHello() 메서드를 가져와서 실행한다
        Method method1 = clazz.getMethod("sayHello");
        method1.invoke(instance); 

        // 4. private 필드 접근
        Field nameField = clazz.getDeclaredField("name");
        nameField.setAccessible(true); // private 필드 접근 허용
        String name = (String) nameField.get(instance);
        System.out.println("name = " + name); 

        // 5. 필드 값 수정
        nameField.set(instance, "new");
    }
}
```

리플렉션을 이용하면 컴파일 당시엔 존재하지 않던 클래스도 이용할 수 있게 된다.

### 리플렉션 단점
- 컴파일타임 타입 검사가 주는 이점을 하나도 누릴 수가 없다. 예외 검사도 마찬가지다.
    - 따라서 리플렉션 기능으로 존재하지 않는 혹은 접근 불가능한 메서드를 호출한다면?
    - 주의해서 대비 코드를 작성해놓지 않았다면 런타임 오류가 발생한다.
    - 개발자가 실수로 잘못된 클래스명이나, 잘못된 인수를 전달해도 런타임에야 발견할 수 있다.
- 리플렉션을 이용하면 코드가 지저분하고 장황해진다.
- 성능이 떨어진다.
    - 리플렉션을 통한 메서드 호출은 일반 메서드 호출보다 훨씬 느리다.

## 리플렉션을 사용해야 할 때 주의
- 코드 분석 도구나 의존 관계 주입 프레임워크처럼 리플렉션이 필요한 애플리케이션이 있긴 하다.
    - 하지만, 이 경우도 리플렉션 사용은 줄이고 있다.
- 리플렉션은 아주 제한된 형태로만 사용해야 그 단점을 피하고 이점만을 취할 수 있다.
    - 컴파일타임에 이용할 수 없는 클래스에 접근하더라도 컴파일타임에 그 인터페이스나 상위 클래스는 존재하는 경우가 일반적이다.
    - 이땐 **리플렉션은 인스턴스 생성에만 쓰고, 이렇게 만든 인스턴스는 인터페이스나 상위 클래스로 참조해 사용하자**
```java
// 리플렉션으로 생성하고 인터페이스로 참조해 활용한다.
public static void main(String[] args) {
    
    // 클래스 이름을 Class 객체로 변환
    Class<? extends Set<String>> cl = null;
    try {
        cl = (Class<? extends Set<String>>) Class.forName(args[0]); //비검사 형변환
    } catch (ClassNotFoundException e) {
        fatalError("클래스를 찾을 수 없습니다.");
    }
    
    // 생성자를 얻는다.
    Constructor<? extends Set<String>> cons = null;
    try {
        cons = cl.getDeclaredConstructor();
    } catch (NoSuchMethodException e) {
        fatalError("매개변수 없는 생성자를 찾을 수 없습니다.");
    }
     
    // 집합의 인스턴스를 만든다.
    Set<String> s = null;
    try {
        s = cons.newInstance();
    } catch (IllegalAccessException e) {
        fatalError("생성자에 접근할 수 없습니다.");
    } catch (InstantiationException e) {
        fatalError("클래스를 인스턴스화할 수 없습니다.");
    } catch (InvocationTargetException e) {
        fatalError("생성자가 예외를 던졌습니다: " + e.getCause());
    } catch (ClassCastException e) {
        fatalError("Set을 구현하지 않은 클래스입니다.");
    }
    
    // 생성한 집합을 사용한다.
    s.addAll(Arrays.asList(args).subList(1, args.length));
    System.out.println(s);
}

private static void fatalError(String msg) {
    System.err.println(msg);
    System.exit(1);
}
```
- 위 코드를 보면 리플렉션은 newInstance() 로 인스턴스를 생성하는 데까지만 사용하였다.
- 다음, 원소를 추가할 땐 직접 set.addAll() 을 사용하여 일반 메서드 호출을 사용하고 있다.
- (Class<? extends Set<String>>) 비검사 형변환 경고
    - 여기서 형변환은 명시한 클래스가 Set 을 구현하지 않았더라도 성공할 것이다.
    - 다만, 클래스 인스턴스를 생성하여 Set 으로 참조하려할 때 ClassCastException 을 던진다.
- 위와 같은 코드는 제네릭 집합 테스터 도구로도 사용할 수 있다. 
    - 즉, 명시한 Set 구현체를 조작해보면서 Set 규약을 잘 지키는지 검사할 수 있다.
- 이와 같은 기법은 완벽한 서비스 제공자 프레임워크를 구현할 수 있을 만큼 강력하다.
- **예제 코드에서 보여지는 리플렉션 단점**
    - 첫째, 런타임에 총 6가지나 되는 예외를 던질 수 있다.
    - 둘째, 클래스 이름만으로 인스턴스를 생성해내기 위해 무려 25줄이나 작성되었다. (리플렉션이 아니라면 한줄이다.)

## 리플렉션 의존성 관리
- 런타임에 존재하지 않을 수도 있는 클래스, 메서드, 필드와의 의존성을 관리할 때 적합하다. (드물긴 하다.)
- 버전이 여러 개 존재하는 외부 패키지를 다룰 때 유용하다.
- 예를 들어 오래된 버전을 사용하는 환경에서 라이브러리의 최신 버전을 사용하고 싶을 때 리플렉션으로 접근하는 방식이다.
    - 하지만, 새로운 클래스나 메서드가 존재하지 않을 수도 있다는 것은 감안해야 한다.
    - 이에 대해선 대체 수단을 이용하거나 기능을 줄여 동작하는 등의 적절한 조치를 취해야 한다.

## 요약 정리
- 리플렉션은 복잡한 특수 시스템을 개발할 때 필요한 강력한 기능이지만 단점이 많다.
- 컴파일타임에 알 수 없는 클래스를 런타임에 사용해야 하는 프로그램을 작성한다면 리플렉션을 이용해야 할 것이다.
- 단, 되도록 인스턴스 생성에만 사용하고, 객체를 이용할 땐 컴파일에 알 수 있는 인터페이스나 상위 클래스로 참조하여 사용하자.