# Item 33. 타입 안전 이종 컨테이너를 고려하라

## 타입 안전 이종 컨테이너 (Type Safe Heterogeneous Container)

- 여러 다양한 타입의 객체들을 하나의 컨테이너에 안전하게 담고 쓸 수 있도록 하는 패턴
- 기존 제네릭 사용과 비교
    - Set<E>, Map<K, V> 등의 컬렉션, `ThreadLocal<T>` 와 같은 단일 원소 컨테이너
    - 여기선 매개변수화되는 대상이 원소가 아닌 컨테이너, 즉 클래스 자신이다.
    - 따라서 하나의 컨테이너에서 매개변수화할 수 있는 타입의 수도 제한되어 있다.
- **컨테이너 대신 키를 매개변수화한 다음, 컨테이너에 값을 넣거나 뺄 때 매개변수화한 키를 함께 제공한다.**
    - 이렇게 하면 제네릭 타입 시스템이 값의 타입이 키와 같음을 보장할 수 있다.
- 예시 코드
    - 타입별로 즐겨 찾는 인스턴스를 저장하고 검색하는 Favorites 클래스
    
```java
public class Favorites {
    // 맵의 키가 비한정적 와일드카드 타입
    // 따라서 여러 타입의 키를 담을 수 있음
    private Map<Class<?>, Object> favorites = new HashMap<>();

    public <T> void putFavorite(Class<T> type, T instance) {
        favorites.put(Objects.requireNonNull(type), instance);
    }

    public <T> T getFavorite(Class<T> type) {
        return type.cast(favorites.get(type));
    }
}
```
```java
public class Main {
    public static void main(String[] args) {
        Favorites f = new Favorites();

        f.putFavorite(String.class, "Java");
        f.putFavorite(Integer.class, 0xcafebabe);
        f.putFavorite(Class.class, Favorites.class);

        String favoriteString = f.getFavorite(String.class);
        int favoriteInteger = f.getFavorite(Integer.class);
        Class<?> favoriteClass = f.getFavorite(Class.class);

        System.out.printf("%s %x %s%n", favoriteString, favoriteInteger, favoriteClass.getName());
    }
}
```
    
- 각 타입의 Class 객체를 매개변수화한 키 역할로 사용
- 컴파일타임 타입 정보와 런타임 타입 정보를 알기 위해 타입 토큰(type token)인 class 리터럴 사용
    - class 리터럴의 타입은 `Class<T>`다.
    - 즉, String.class의 타입은 `Class<String>`이다.
    - 여러 class 객체를 키로 받을 수 있는 것은 class의 클래스가 제네릭이기 때문에 가능하다.
- Map의 키는 비한정적 와일드카드 타입으로 어떤 타입도 저장할 수 있다.
- Map의 값 타입은 단순히 Object이다.
    - 이 맵은 모든 값이 명시한 키의 타입임을 보증하지 않는다.
    - 그러나, 꺼낼 때 매개변수화한 키를 같이 제공하므로 관계가 성립함을 우린 알고 있다.
    - 조회 시 Class의 cast 메소드를 사용해 Class 객체가 가리키는 타입으로 동적 형변환한다.
        - cast 메소드?
        - 단순히 주어진 인수가 Class 객체가 알려주는 타입의 인스턴스인지를 검사한 다음, 맞다면 인수를 그대로 반환하고, 아니면 ClassCastException을 던진다.
        
___

### 제약 사항

- 악의적인 클라이언트가 Class 객체를 로 타입으로 넘기면 타입 안전성이 쉽게 깨진다.
    - 로 타입으로 캐스팅하여 넘김
    
    ```java
    f.putFavorite((Class) Integer.class, "hi");
    int x = f.getFavorite(Integer.class); // ClassCastException
    ```
    
    - 동적 형변환으로 런타임 타입 안전성 확보
    
    ```java
    public <T> void putFavorite(Class<T> type, T instance) {
        favorites.put(Objects.requireNonNull(type), type.cast(instance));
    }
    ```
    
    - Favorites가 타입 불변성을 어기는 일이 없도록 보장한다.
    - map 에 put 할 때 인수로 주어진 인스턴스를 cast 하여 타입을 확인한다.
    - java.util.Collections의 checkedSet, checkedList 등의 메소드가 그 예시이다.
    - 제네릭과 로 타입을 섞어 사용하는 애플리케이션에서 잘못된 타입의 원소를 넣지 못하게 도움을 준다.
- 실체화 불가 타입에는 사용할 수 없다.
    - 즉, `List<String>` 타입은 map의 키로 저장할 수 없다.
    - `List<String>`용 Class 객체를 얻을 수 없기 때문이다. 즉, `List<String>.class`는 불가하다.
    - `List<T>` → 모두 같은 List.class라는 같은 Class 객체를 사용한다.
    - 해결 방법
        - 슈퍼 타입 토큰(super type token)이 존재한다.
        - 닐 개프터(Neal Gafter)가 고안한 방식이다.
        - 실제로 유용하여 스프링에선 아예 `ParameterizedTypeReference`클래스로 미리 구현해놓았다.
        - 그러나 이 방식도 완벽하진 않아 완벽한 우회는 없다.
        
        ```java
        f.putFavorites(new TypeRef<List<String>>(){}, pets);
        ```
        
**`ParameterizedTypeReference<T>`**
- org.springframework.core.ParameterizedTypeReference<T>
- 제네릭 타입을 런타임에 안전하게 전달하기 위한 타입 토큰이다.
- 주로 RestTemplate 과 사용하여 외부 API로부터 받은 응답을 어떤 Object로 변환할지 명시한다.
- 즉, Http 요청 처리에 사용한다.
- TypeReference<T> 과 목적은 동일하나 이 클래스는 Spring 에서 제공한다.
    - TypeReference<T> 는 ObjectMapper와 사용한다.

```java
ResponseEntity<List<User>> response = restTemplate.exchange(
    "/users",
    HttpMethod.GET,
    null,
    new ParameterizedTypeReference<List<User>>() {} // 응답 반환 타입 명시
);
```
___

### 한정적 타입 토큰 사용

- 위 Favorites 예시의 경우 타입 토큰이 비한정적이므로 어떤 Class 객체든 받아들인다.
- 한정적 타입 토큰은 타입을 제한하고 싶을 때 사용하면 된다.
- Annotation API 은 한정적 타입 토큰을 적극 사용한다.
    
```java
package java.lang.annotation;

public interface Annotation {
    boolean equals(Object obj);
    int hashCode();
    String toString();
    Class<? extends Annotation> annotationType();
}
```

```java
// AnnotationElement 인터페이스에 선언된 메소드
public <T extends Annotation> T getAnnotation(Class<T> annotationType);
```

- 대상 요소에 달려 있는 어노테이션을 런타임에 읽어 오는 기능을 한다.
- 리플렉션의 대상이 되는 타입들, 즉 클래스(java.lang.Class<T>), 메서드(java.lang.reflect.Method), 필드(java.lang.reflect.Field) 같이 프로그램 요소를 표현하는 타입들에서 구현한다.
- annotationType 인수는 어노테이션 타입을 뜻하는 한정적 타입 토큰이다.
- 위 메소드는 토큰으로 명시한 타입의 어노테이션이 대상에 달려있다면 반환하고, 없다면 null 반환한다.
- 즉, 어노테이션된 요소는 그 키가 어노테이션 타입인 타입 안전 이종 컨테이너인 것이다.

**여기서 Class<?>와 같이 비한정적 와일드카드 타입을 한정적 타입 토큰을 받는 메서드에 전달하려면?**

- Class<? extends Annotation>으로 형변환할 수 있지만 비검사 형변환 경고가 뜬다.
- Class에서는 이러한 동적 형변환을 안전하게 수행해주는 asSubclass 메서드를 제공한다.
    
```java
public <U> Class<? extends U> asSubclass(Class<U> clazz) {
    if (clazz.isAssignableFrom(this))
        return (Class<? extends U>) this;
    else
        throw new ClassCastException(this.toString());
}
```
- Class<? extends U> 반환
- 호출된 인스턴스 자신의 Class 객체를 인수가 명시한 클래스로 형변환한다.
- 형변환이 된다는 것은 자신이 인수로 명시한 클래스의 하위 클래스라는 것이다.
- 형변환에 성공한다면 인수로 받는 클래스 객체를 반환, 실패하면 ClassCastException을 던진다.
- asSubclass 사용하는 예시
    
```java
static Annotation getAnnotation(AnnotatedElement element, String annotationTypeName) {
    Class<?> annotationType = null; // 비한정적 타입 토큰
    
    try {
        annotationType = Class.forName(annotationTypeName); // 어노테이션 클래스 이름 로딩
    } catch (Exception ex) {
        throw new IllegalArgumentException(ex);
    }
        
    return element.getAnnotation(annotationType.asSubclass(Annotation.class));
}
```
    
```java
Class<?> raw = Class.forName("com.example.MyAnno");
Class<? extends Annotation> annotationType = raw.asSubclass(Annotation.class);
Annotation ann = element.getAnnotation(annotationType);
```
    
- 컴파일시점엔 타입을 알 수 없는 어노테이션을 런타임에 읽어내는 예이다.
- MyAnno 이 Annotation 의 서브 클래스임을 알려준다.

## 요약 정리


- 일반적인 제네릭 형태에선 한 컨테이너가 다룰 수 있는 타입 매개변수의 수가 고정되어 있다.
- 컨테이너 자체가 아닌 키를 타입 매개변수로 바꾸면 제약 없는 타입 안전 이종 컨테이너를 만들 수 있다.
- 타입 안전 이종 컨테이너는 Class 객체를 키로 쓰며, 이것을 타입 토큰이라고 한다.