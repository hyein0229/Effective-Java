# Item 27. 비검사 경고를 제거하라

## 비검사 경고 (Unchecked Warning) 란?

- 제네릭을 사용할 때 **컴파일러가 타입 안정성을 확인하는데 필요한 정보가 충분치 않을 때 발생하는 경고**이다.
- 제네릭을 사용하면 수많은 컴파일러 경고를 보게 된다.
- 비검사 경고는 쉽게 제거할 수 있다.
- **타입 안전성 보장을 위해 할 수 있는 한 모든 비검사 경고를 제거하라.**
    - 즉, 런타임에 ClassCastException이 발생하지 않고 모두 의도대로 동작할 것이다.
- javac -Xlint: uncheck 옵션을 추가하면 컴파일러가 무엇이 잘못됐는지 설명해준다.

## 컴파일러의 경고를 보고 제거해라

```java
Set<Coin> coinSet = new HashSet();
```

- 제네릭 클래스인데 타입 매개변수를 사용하지 않아 unchecked conversion 경고가 발생한다.
- 이는 컴파일러가 알려준 타입 매개변수를 명시하지 않고, 다이아몬드 연산자<>만으로 해결할 수 있다.

```java
Set<Coin> coinSet = new HashSet<Coin>();
Set<Coin> coinSet = new HashSet<>(); // 컴파일러의 추론 기능 활용
```

- 다이아몬드 연산자는 컴파일러가 올바른 실제 타입 매개변수를 추론해준다.

## 경고를 제거할 순 없으나 안전하다면 경고를 숨기자

- 경고를 제거할 순 없지만 타입 안전하다고 확신할 수 있다면 **@SuppressWarnings(”unchecked”)** 어노테이션을 달아서 경고를 숨기자.
- 단, 타입 안전성은 검증을 한 후 숨겨야 함에 주의하자.
- 한편, **안전하다고 검증된 비검사 경고를 숨기지 않으면 진짜 문제가 발생했을 때 새로운 경고가 나와도 눈치를 채지 못할 수 있다.**
    - 제거하지 않은 거짓 경고 속 새로운 진짜 문제의 경고가 묻힐 수 있다.

### **@SuppressWarnings(”unchecked”)**

- 개별 지역변수 선언부터 클래스 전체까지 어떤 선언에도 달 수 있다.
- 하지만, **항상 가능한 한 좁은 범위에 적용하자.**
- 보통은 변수 선언, 아주 짧은 메소드, 혹은 생성자에 사용한다.
- 자칫 심각한 경고를 놓칠 수 있으므로 클래스 전체와 같이 넓은 범위에 적용해선 안된다.
- 한 줄이 넘는 메소드나 생성자에 달린 어노테이션을 발견하면 지역변수 선언쪽으로 옮겨라
    - 이를 위해서 새로운 지역 변수를 선언해야 할 수 있지만 그만한 가치가 있다.
    
```java
// ArrayList 의 toArray 메소드
public <T> T[] toArray(T[] a) {
    if (a.length < size)
        // Make a new array of a's runtime type, but my contents:
        return (T[]) Arrays.copyOf(elementData, size, a.getClass());
    System.arraycopy(elementData, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```
    
```bash
    return (T[]) Arrays.copyOf(elementData, size, a.getClass());
```
    
- 이때, 위 코드에서 unchecked cast, 즉 확인되지 않는 형변환 경고가 발생한다.
- 어노테이션은 선언에만 달 수 있으므로 return 문에는 달 수 없다.
- 그렇다면 메소드에 달아야 할까? 메소드 전체는 범위가 너무 넓어지므로 자제해야 한다.
- 대신, **반환값을 담을 지역변수를 하나 선언하고 그 변수에 어노테이션을 달아주자.**
    
```java
public <T> T[] toArray(T[] a) {
    if (a.length < size) {
        
        // 생성한 배열과 매개변수로 받은 배열의 타입이 모두 T[]로 같으므로 올바른 형변환이다.
        @SuppressWarnings("unchecked")
        T[] result = (T[]) Arrays.copyOf(elementData, size, a.getClass());
        return result;
    }
    System.arraycopy(elementData, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```
    
- **그리고 어노테이션을 달 땐 그 경고를 무시해도 안전한 이유를 항상 주석으로 남겨야 한다.**
    - 다른 사람이 그 코드를 볼 때 이해를 돕는다.
    - 코드를 잘못 수정하여 타입 안전성을 잃는 상황을 방지한다.
    

### **SuppressWarnings 옵션**

| all | 모든 경고 |
| --- | --- |
| cast | 캐스트 연산자 경고 |
| dep-ann | 사용하지 말아야할 주석 경고 |
| deprecation | 사용하지 말아야할 메서드 경고 |
| fallthrough | switch 문 break 누락 경고 |
| finally | 반환하지 않은 finally 블럭 경고 |
| null | null 경고 |
| rawtypes | 제네릭을 사용하는 클래스 매개변수가 불특정일때 경고 |
| unchecked | 검증되지 않은 연산자 경고 |
| unused | 사용하지 않은 코드 관련 경고 |

## 요약 정리

- 비검사 경고(uncheked warnings)는 중요하므로 무시하면 안된다.
- 비검사 경고는 런타임에 ClassCastException을 일으킬 수 있는 잠재적 가능성을 뜻하므로 제거해라.
- 경고를 없앨 방법을 찾지 못했다면 타입 안정성을 검사하고, 안전하다면 경고를 숨겨라
- @SuppressWarnings(”unchecked”) 는 가능한 한 범위를 좁혀 적용해라