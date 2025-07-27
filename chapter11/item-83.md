# Item 83. 지연 초기화는 신중히 사용하라
## 지연 초기화(lazy initialization)
- 필드의 초기화 시점을 그 값이 처음 필요할 때까지 늦추는 기법
- 따라서 값이 전혀 쓰이지 않으면 초기화도 일어나지 않는다.
- 이점
    - 성능 최적화 용도
    - 클래스와 인스턴스 초기화 시 발생하는 위험함 순환 문제 해결 효과 
- **하지만, 지연 초기화는 필요할 때까지는 하지 마라**
    - 고려할 것이 많다.
    - 결국 초기화가 이루어지는 비율, 초기화에 드는 비용에 따라, 초기화된 필드를 얼마나 호출하는가에 따라 성능이 달라진다.
    - 즉, 실제로는 성능을 저하시킬 수 있다.
- 지연 초기화가 필요할 때는?    
    - 해당 클래스의 인스턴스 중 그 필드를 사용하는 인스턴스의 비율이 낮은 반면, 초기화 비용이 큰 경우
    - 하지만, 이것이 확실히 최적화를 해주냐는 직접 전후 성능을 측정해보는 수밖에 없다.
- 멀티스레드 환경에서의 지연 초기화는 주의해야 한다.
    - 지연 초기화는 그 필드를 둘 이상의 스레드가 공유한다면 무조건 동기화해야 한다.
    - 동기화하지 않으면 심각한 버그로 이어질 것이다.
- **대부분 상황에선 일반 초기화가 지연 초기화보다 낫다.**

## 지연 초기화 방법
### synchronized 지연 초기화
```java
// 일반적인 인스턴스 필드 초기화
private final FieldType field = computeFieldValue();
```
```java
// 지연 초기화 - synchronized 방식
private FieldType field;

private synchronized FieldTYpe getField(){
    if(field == null)
        field = computeFieldValue();
    return field;
}
```
- 초기화 순환성 문제를 깨트릴 것 같으면 synchronized 를 단 접근자를 사용해라
- synchronized 는 한 스레드만 접근할 수 있도록 해주어 하나의 인스턴스만 생성될 수 있도록 한다. 
    - 스레드 안전성 보장

___

### 정적 필드 초기화 - 지연 초기화 홀더 클래스 사용
```java
// lazy initialization holder class 
private static class FieldHolder{
    static final FieldType field = computeFieldValue();
}

private static FieldType getField() {return FieldHolder.field;}
```
- 싱글턴 구현 시 했던 구조
- getField() 가 처음 호출되는 순간 FieldHolder.field 가 읽히면서 FieldHolder 클래스를 로딩하고 초기화한다.
- 장점은 필드에 접근하면서 동기화를 전혀 하지 않으므로 성능이 느려질 거리가 없다는 것이다.
- 일반 jvm 은 오직 클래스를 초기화할 때만 필드 접근을 동기화한다.
    - 내부적으로 초기화 시에 동기화 수행
    - 클래스 초기화가 끝나면 jvm 이 동기화 코드를 제거
    - 따라서 초기화 이후엔 어떤 동기화 없이 필드에 접근 가능

___

### 이중검사(doble-check) 관용구 사용
```java
private volatile FieldType field;

private FieldType getField() {
    FieldType result = field;
    if (result != null) // 첫 번째 검사 (락 사용 안 함)
        return result;

    synchronized (this) {
        if (field == null) // 두 번째 검사 (락 사용)
            field = computeFieldValue();
        return field;
    }
}
```
- 초기화된 이후에는 동기화를 없애어 준다.
- 즉, 처음 한 번은 동기화 없이 검사한 후, 아직 초기화되지 않았다면 두 번째는 동기화하여 검사한다.
- 두 번째에서도 초기화되지 않은 경우에만 필드를 초기화한다.
- 필드가 초기화된 이후엔 동기화하지 않으므로 필드를 volatile 로 선언해야 한다.
- 하지만, 정적 필드에는 지연 초기화 홀더 클래스 방식이 더 낫다.

___

### 단일검사(single-check) 관용구
```java
private FieldType getField() {
    FieldType result = field;
    if (result == null) // 첫 번째 검사 (락 사용 안 함)
        field = result = computeFieldValue();  
    return result;
}
```
- 만약, 초기화가 여러번 되어도 상관없는 필드라면 이중검사에서 두 번째 검사를 생략할 수 있다.
- 스레드 안전하지 않으므로 초기화가 중복으로 일어날 수 있음에 주의해야 한다.
    - 따라서 초기화 비용이 크지 않는 경우에 적합하다. 
- long 과 double 을 제외한 기본 타입이라면 volatile 도 없애도 된다.
    - 기본 타입은 읽기, 할당에서 원자적 수행때문에?
    - 하지만, 거의 쓰이지 않는다.

## 요약 정리 
- 대부분의 필드는 일반적으로 초기화하고 지연 초기화하지 마라
- 만약 성능 개선이나 초기화 순환 문제를 막기 위해 지연 초기화해야 한다면 올바르게 사용해라
- 일반 인스턴스 필드에는 이중검사 관용구를, 정적 필드엔 지연 초기화 홀더 클래스(holder)를 사용해라
- 반복해 초기화해도 괜찮은 인스턴스 필드는 단일검사 관용구를 고려해도 괜찮다.