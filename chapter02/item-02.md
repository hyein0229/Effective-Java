# Item 02. 생성자에 매개변수가 많다면 빌더를 고려하라

생성자와 정적 팩토리 메소드 방식은 선택적 매개변수가 많아질수록 대응하기 어려워진다는 제약이 존재한다. 매개변수가 많을 때 **클래스를 인스턴스화할 수 있는 방법에는 ‘점층적 생성자 패턴’, ‘자바빈즈 패턴’, ‘빌더 패턴’ 3가지가 존재한다.**

## 점층적 생성자 패턴 (telescoping constructor pattern)

필수 매개변수만 받는 생성자, 선택 매개변수 1개를 받는 생성자, 2개 받는 생성자.. 형태로 선택 매개변수를 모두 받는 생성자까지 점층적으로 늘려가는 방식이다.

```java
public class Member {
	
    private String name;				// 필수
    private int age;				   	// 필수
    private String address;				// 선택
    private String phone;				// 선택
    private String email;  				// 선택
    
    // 필수 매개변수를 가지는 생성자
    public Member(String name, int age) {
    	this(name, age, null, null, null);
    }
    
    // 선택 매개변수 address가 추가된 생성자
    public Member(String name, int age, String address) {
    	this(name, age, address, null, null);
    }
    
    // 선택 매개변수 phone이 추가된 생성자
   	public Member(String name, int age, String address, String phone) {
    	this(name, age, address, phone, null);
    }
    
    // 모든 매개변수를 가지는 생성자
    public Member(String name, int age, String address, String phone, String email) {
    	this.name = name;
      this.age = age;
      this.address = address;
      this.phone = phone;
      this.email = email;
    }
}
```

**단점**

- 매개변수가 많아질수록 생성자가 늘어나고 코드 작성 효율과 가독성이 저하된다.
- 클라이언트 코드를 작성하거나 읽기가 어려워진다.
    - 사용자가 설정하길 원치 않는 매개변수를 포함하게 되는 경우가 많아진다.
    - 코드를 읽을 때 각 값의 의미가 무엇인지, 매개변수가 몇 개인지 주의해야 한다. 즉, 매개변수를 넘길 때 실수가 잦아질 수 있다.
    - 타입이 같은 매개변수가 연달아 있을 경우 찾기 어려운 버그로 이어질 수 있다. (컴파일러로 찾을 수 없는 문제이기 때문이다.)

## 자바빈즈 패턴(JavaBeans Pattern)

*매개변수가 없는 생성자로 객체를 만든 후, Setter 메소드를 호출하여 원하는 매개변수의 값을 설정하는 방식이다.*

```java
Member member = new Member();
member.setName("jan");
member.setAge(99);
member.setAddress("KOR");
member.setPhone("01012345678");
```

**장점**

- 코드가 길지만 인스턴스 생성이 쉽고, 가독성이 좋아진다.
    - 어떤 데이터에 어떤 값을 주입하는지 한눈에 파악 가능하다.
    - 연속된 동일 타입 매개변수 설정에서 점층적 생성자 방식과 다르게 실수를 막을 수 있다.
- 필요한 매개변수만을 Setter 를 통해 주입할 수 있다.
- 디폴트 매개변수 주입의 생략을 간접적으로 지원한다.
    - 빌더 클래스 내부에서 디폴트 값을 설정한다.
    - 빌더 클래스를 경유하여 디폴트 매개변수가 설정된 필드 메소드를 호출하지 않을 수 있다.
- 필수 매개변수는 빌더의 생성자로, 선택 매개변수는 set 메소드로 분리할 수 있다.
- 멤버 변수 초기화 시 유효성 검증을 멤버별로 분리 가능하다.

**단점**

- 객체 하나를 만들기 위해 메소드를 여러 개 호출해야 한다.
- 객체가 완전히 생성되기 전까진 일관성(consistency)이 무너진 상태에 놓인다.
    - 점층적 생성자 패턴의 경우 매개변수의 유효성만 생성자에서 확인하면 일관성 유지가 가능하다.
- Setter 메소드로 인해 불변의 객체(immutable)를 만들 수 없다.
    - 외부적으로 Setter 메소드가 노출되어 조작 가능성이 있기 때문이다.
- 즉, 객체 생성 시점에 모든 값이 주입되지 않아 **일관성**과 **불변성** 문제가 나타난다.

## 빌더 패턴(Builder Pattern)

*점층적 생성자 패턴의 ‘안전성’과 자바빈즈 패턴의 ‘가독성’을 겸비한 빌더 패턴이다.* 

- 필요한 객체를 직접 만들지 않고 필수 매개변수만으로 생성자(혹은 정적 팩토리 메소드)를 호출하여 빌더 객체를 얻는다.
- 일종의 세터 메소드로 원하는 매개변수만을 설정한다.
- 마지막으로 build 메소드 호출로 필요한 객체를 얻게 된다.

빌더는 생성할 클래스 안에 **정적 멤버 클래스 (static inner class)** 로 만들어두는 것이 일반적이다.

**why?**

- 빌더를 내부 클래스로 만들어 그룹핑함으로써 관계를 쉽게 파악한다.
- 생성자를 외부에 노출하지 않기 위해 private 으로 선언하고 빌더 내부에서만 호출하기 위함이다.
- static 인 이유는 내부 클래스를 외부 클래스를 인스턴스화하지 않고도 생성하기 위함이다.
- static 은 ‘외부 참조’가 존재하지 않으므로 GC 수거 대상이 되어 메모리 누수 문제가 발생하지 않는다.
    - 내부 클래스에서 외부 클래스의 값을 사용하지 않는다면 static inner 클래스로 선언해주어야 한다.

```java
public class Member {

    private String name;
    private int age;
    private String address;
    private String phone;
		
		// Builder 객체를 통해 인스턴스의 매개변수 값 설정
		// 반드시 private 으로 선언하여 외부 호출을 방지한다.
    private Member(Builder builder) {
        this.name = builder.name;
        this.age = builder.age;
        this.address = builder.address;
        this.phone = builder.phone;
    }

    // 빌더 호출, 외부에서 Member.builder() 으로 접근할 수 있도록 정적 메소드 생성
    public static Builder builder() {
        return new Builder();
    }

    // static inner class 생성
    public static class Builder {
	      // 모두 선택 필드라고 할 때
        private String name;
        private int age;
        private String address;
        private String phone;
				
				// 이때 만약, 필수 매개변수가 있다면 생성자에서 받는다.
        private Builder() {};
				
				// 선택 매개변수에 대해 setter 메소드 제공
        public Builder name(String name) {
            this.name = name;
            return this; // 빌더 자신을 반환하여 메소드 체이닝이 가능함
        }

        public Builder age(int age) {
            this.age = age;
            return this;
        }

        public Builder address(String address) {
            this.address = address;
            return this; 
        }

        public Builder phone(String phone) {
            this.phone = phone;
            return this;
        }

        // 마지막에 build 메소드를 실행하여 객체를 생성하고 return 하도록 구현
        public Member build() {
            return new Member(this);
        }
    }
}
```

```java
Member member = Member.builder()
                .name("jan")
                .age(99)
                .address("jan")
                .phone("010")
                .build();
```

- Member.builder → Member 의 빌더 객체를 반환
- 세터 메소드를 통해 빌더에 매개 변수를 설정
- 메소드 체이닝을 통해 연쇄적으로 세터 메소드 호출
- 마지막 build 호출하여 Member 객체를 얻음

**특징**

- 세터 메소드에서 자신을 반환하도록 하여 연쇄적 호출이 가능하다.
    - 이를 **플루언트 API(fluent API) 혹은 메소드 연쇄(method chaining)** 이라고 한다.
- 계층적으로 설계된 클래스와 함께 쓰기가 좋다.

**장점**

- 코드를 쓰기 쉽고 가독성이 좋다.
- 가변인수 매개변수를 여러 개 사용이 가능하다.
- 넘기는 매개변수에 따라 서로 다른 객체를 쉽게 만들 수 있어 유연하다.
- 어느 필드에 어떤 값을 채워야 하는 지 명확하게 알 수 있다.
- 클래스의 불변성을 유지할 수 있다.

**단점**

- 빌더 생성 비용이 크진 않으나 성능에 민감한 상황에선 문제가 될 수 있다.
- 코드가 더 장황해져 매개변수가 4개 이상으로 많을 때만 값어치를 한다.


API는 시간이 지날수록 매개변수가 많아지는 경향이 있다. 
따라서, 초기엔 생성자나 정적 팩토리 패턴 방식으로 시작하여 빌더 패턴으로 바꿀 수 있지만 애초에 빌더로 시작하는 것이 나을 수 있다.


### 계층적으로 설계된 클래스에서의 빌더 패턴

---

추상 클래스(상위 클래스)에서는 추상 빌더를, 구체 클래스(하위 클래스)에서는 구체 빌더를 갖는다.

Pizza 의 하위 계층

- NyPizza : 크기 매개변수를 필수로 받는다.
- Calzone : 소스를 안에 넣을지 선택하는 매개변수를 필수도 받는다.

```java
public abstract class Pizza{
    public enum Topping { HAM, MUSHROOM, ONION, PEPPER, SAUSAGE }
    final Set<Topping> toppings;
    
    abstract static class Builder<T extends Builder<T>> {
       EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);
       
       // 하위 클래스 에서 공통으로 사용할 토핑 
       public T addTopping(Topping topping) {
          toppings.add(Objects.requireNonNull(topping));
          return self();
       }
 
       // Pizza 를 상속받은 클래스 ( NyPizza , Calzone ) return 
       abstract Pizza build();
 
       // 하위 클래스는 이 메서드를 재정의하고 this 를 반환하도록 한다.
       protected abstract T self();
    }
 
    // 저장된 토핑이 build()를 호출할 때 각각 생성자에서 super(builer)를 호출하고
    // super는 Pizza의 생성자 이므로 아래의 Pizza가 호출 되면서 토핑을 clone()해서 저장한다.
    Pizza(Builder<?> builder) {
       toppings = builder.toppings.clone();
    }
 }
```

1번 : Pizza 의 기본 생성자는 존재하지 않으며 Builder 를 매개변수로 받는 생성자만 존재한다.

2번 : 1번으로 인해 Pizza 하위 계층에선 무조건 생성자에서 Pizza 의 Builder 를 넘겨줘야 한다.

3번 : 2번으로 인해 Pizza 하위 계층에선 Pizza 의 Builder 를 상속받는 Builder 클래스를 만들어야 한다.

Pizza 하위 클래스

```java
public class NyPizza extends Pizza {
    public enum Size { SMALL, MEDIUM, LARGE }
    private final Size size;
		 
		// Pizza 의 Builder 를 상속받는 Builder 구현
    public static class Builder extends Pizza.Builder<Builder> {
       private final Size size;
 
       // 필수 매개변수를 받는 Builder 생성자
       public Builder(Size size) {          
					super();
          this.size = Objects.requireNonNull(size);
       }
 
       @Override 
			 public NyPizza build() {
          return new NyPizza(this);
       }
 
       @Override 
			 protected Builder self() { return this; }
    }
 
    private NyPizza(Builder builder) {
       super(builder); // 상위 Pizza 클래스에 Builder 넘겨 호출
       size = builder.size;
    }
 }

public class Calzone extends Pizza {
    private final boolean sauceInside;
 
    public static class Builder extends Pizza.Builder<Builder> {
       private boolean sauceInside = false;
 
       public Builder() {          
         super();
       }
 
       // sauceInside 는 옵션 
       public Builder sauceInside() {
         sauceInside = true;
         return this;
      }
 
       @Override 
       public Calzone build() {
          return new Calzone(this);
       }
 
       @Override 
       protected Builder self() { return this; }
    }
 
    private Calzone(Builder builder) {
       super(builder);
       sauceInside = builder.sauceInside;
    }
 
 }
```

### Lombok 의 @Builder 어노테이션

---

```java
@Builder
public class Member {
    private String username;
    private int age;
}
```

@Builder 어노테이션은 기본적으로 생성자나 메소드 레벨에 적용할 수 있다. 하지만, 클래스에도 적용은 가능한데 클래스에 적용하는 경우 모든 매개변수를 받는 package-private 생성자 (default 접근 제어자) 가 자동으로 생성되고 해당 생성자에 @Builder 어노테이션이 붙은 것처럼 동작한다.

다음은 @Builder 어노테이션으로 인해 실제 컴파일된 후 생성된 코드이다. 

```java
public class Member {
    private String username;
    private int age;
		
		// default 생성자 자동 생성
    Member(String username, int age) {
        this.username = username;
        this.age = age;
    }
		
		// 빌더 생성
    public static Builder builder() {
        return new Builder();
    }
		
		// static inner class 로 빌더 클래스 자동 생성
    public static class Builder {
        private String username;
        private int age;

        Builder() {}

        public Builder username(String username) {
            this.username = username;
            return this;
        }

        public Builder age(int age) {
            this.age = age;
            return this;
        }

        public Member build() {
            return new Member(this.username, this.age);
        }
        
        public String toString() {
            return "Member.Builder(username=" + this.username + ", age=" + this.age + ")";
        }
        
    }
}
```

## 요약 정리

- 생성자나 정적 팩토리 메소드가 처리해야 할 매개변수가 많다면 빌더 패턴 선택이 낫다.
- 매개변수 중 다수가 필수가 아니거나 같은 타입이라면 특히 빌더 패턴이 유용하다.
- 빌더 패턴은 클라이언트 코드 작성이 훨씬 간결하고 가독성이 좋으며 자바빈즈보다 훨씬 안전하다.