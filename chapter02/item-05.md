# Item 05. 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라

## 싱글턴 / 유틸리티 클래스의 문제점

**문제점**

- 상태를 가지게 되는 경우 멀티스레딩 환경에서 버그가 일어나 사용할 수가 없다.
- 테스트를 하기가 어렵다.
- 객체가 스스로 가지고 있는 자원을 가지고 행동을 한다는 점에서 객체 지향 개념과는 멀어진다.

___

### 정적 유틸리티를 잘못 사용한 예

철자 검사기 유틸리티 클래스 

```java
public class SpellChecker {
        // 하나의 고정된 사전을 가진다. -> 유연하지 못하다.
		private static final Lexicon dictionary = ...;

		private SpellChecker() {} 
		
		// 정적 메소드만을 가진 유틸리티
		public static boolean isValid(String word) { ... }
		public static List<String> suggestions(String typo) { ... }
}
```

___

### 싱글턴을 잘못 사용한 예


```java
public class SpellChecker {
        // 여기서도 하나의 고정된 사전을 가짐
		private final Lexicon dictionary = ...;

		private SpellChecker(...) {}
		
        // 내부에서 직접 객체 생성 -> 테스트 어려움
		// 싱글턴 public static 필드 방식
		public static INSTANCE = new SpellChecker(...);

		public boolean isValid(String word) { ... }
		public List<String> suggestions(String typo) { ... }
}
```

사전을 참고하여 철자 검사를 해주는 클래스이다. 두 방식 모두 하나의 사전만을 사용한다고 가정한다는 점에서 좋은 코드가 아니다. 실전에선 여러 언어 별로 사전이 있을 수 있기 때문에 인스턴스 생성 없이 static 으로 직접 객체를 만들어 사용하는 경우 사전을 추가하기가 쉽지 않기 때문이다. 그렇다고 사전을 추가하는 setDictionary 와 같은 메소드를 추가하면 멀티스레딩 환경에선 버그가 일어나기 쉽다. <br>
따라서 클래스가 여러 자원 인스턴스를 지원하고 내부 멤버 인스턴스에 의해 다른 행동을 해야 하는 인스턴스는 싱글턴이나 유틸리티 클래스가 아닌 ***의존성 주입 방식***으로 구현하는 것이 좋다.

## 의존 객체 주입 방식 (의존성 주입 방식)

```jsx
public class SpellChecker {

		private final Lexicon dictionary;
		
		// 생성자에서 의존 객체 주입
		public SpellChecker(Lexicon dictionary) {
			this.dictionary = Objects.requireNonNull(dictionary);
		}

		public boolean isValid(String word) { ... }
		public List<String> suggestions(String typo) { ... }
}
```

- 의존 객체 주입 방식은 유연성과 테스트 용이성을 개선한다. 
- 또한, 여러 객체가 동일 객체를 공유하여(스프링의 기본 싱글톤 방식 의존 주입) 참조도 가능해진다. 
- 하지만 문제점은 의존 객체가 많아질수록 코드가 어지러워질 수 있는데  Dagger, Guice, Spring 과 같은 다양한 의존성 관리 프레임워크는 이러한 문제를 개선해줄 수 있다.

___

### 의존성 주입 방식의 좋은 활용 - 팩토리 주입

---

```jsx
Mosaic create(Supplier<? extends Tile> tileFactory) { ... } 
```

Supplier<T>를 이용해 명시한 타입의 하위 타입이라면 무엇이든 생성할 수 있는 팩토리를 의존 객체로 넘겨 더욱 유연한 객체 생성을 지원 가능하다. 위 코드는 모자이크 객체를 만들기 위해 타일 객체 팩토리를 주입하여 사용하는 코드이다. 

Optional.orElseGet(Supplier<? extends T>)

```java
// Optional 결과가 비어있을 때, 새로운 유저 생성
// 지연 생성의 대표적인 사례
Optional<User> maybeUser = userRepo.findById(id);
User user = maybeUser.orElseGet(() -> new User("default"));
```

## 핵심 정리

- 클래스 내부에서 하나 이상의 자원에 의존하고, 자원이 클래스 동작에 영향을 준다면 싱글턴과 정적 유틸리티 클래스 사용은 지양해라.
    - 자원을 클래스가 직접 만들게 해서도 안된다.
- 필요한 자원 (혹은 자원의 팩토리) 를 생성자에 넘겨 의존 객체를 주입해주자.
- 의존 객체 주입 방식은 유연성, 재사용성, 테스트 용이성을 기막히게 개선한다.