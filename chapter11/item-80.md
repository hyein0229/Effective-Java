# Item 80. 스레드보다는 실행자, 태스크, 스트림을 애용하라
## 실행자 프레임워크(Executor Framework)
- 이전에는 작업큐 구현 시 응답 실패나 응답 불가 문제를 막기 위한 코드를 직접 구현해야 했다.
- `java.util.concurrent` 패키지가 등장하면서 Executor 라고 하는 인터페이스 기반의 유연한 태스크 실행 기능을 제공한다.
```java
// 작업큐 생성
ExecutorService exec = Executors.newSingleThreadExecutor();

// 어떤 태스크를 수행할 지 실행자에게 넘긴다.
exec.execute(runnable);
executor.execute(() -> { // 람다로 넘기는 예시
    System.out.println("Task executed");
}); 

// 실행자를 종료시킨다. (이 작업이 실패하면 VM 자체가 종료되지 않을 것이다)
exec.shutdown();
```

**실행자 서비스의 주요 기능**
- 특정 태스크가 완료되기를 기다리고 결과를 반환받는다. get()
- 태스크 모음 중 아무것 하나(invokeAny()) 혹은 모든 태스크(invokeAll())가 완료되기를 기다린다. 
- 실행자 서비스가 종료하기를 기다린다. awaitTermination()
- 완료된 태스크들의 결과를 차례로 받는다. ExecutorCompletionService 이용
- 태스크를 특정 시간에 혹은 주기적으로 실행하게 한다. ScheduledThreadPoolExecutor 이용

큐를 둘 이상의 스레드가 처리하게 하고 싶다면 Executors 정적 팩터리를 사용하면 된다.
```java
// 스레드 1개만 생성
ExecutorService executor = Executors.newSingleThreadExecutor();

// 고정된 개수, 30개 스레드 풀 생성
ExecutorService exec = Executors.newFixedsThreadExecutor(30);

// 필요할 때마다 스레드 생성 (개수가 제한되지 않음)
// 이전 생성된 스레드 재사용
// 유휴 스레드는 60초 이후 제거
ExecutorService executor = Executors.newCachedThreadPool();
```
- 보통 필요한 실행자 대부분은 Executors 정적 팩터리를 이용해 생성할 수 있다. 
- 하지만 평범하지 않은 실행자(커스텀?)을 원한다면 ThreadPoolExecutor 클래스를 직접 사용할 수 있다. 
- ThreadPoolExecutor 는 스레드 풀 동작을 결정하는 거의 모든 속성을 설정할 수 있다.

___

### Executors.newCachedThreadPool
- 작은 프로그램이나 가벼운 서버라면 CachedThreadPool 사용이 일반적으로 좋은 선택일 것이다.
- 특별히 설정할 것이 없고 일반적인 용도에 적합하게 동작한다.
- 하지만 CachedThreadPool 는 무거운 프로덕션 서버엔 좋지 못하다.
    - CachedThreadPool 은 요청받은 태스크들이 큐에 쌓이지 않고 즉시 스레드에 위임되어 실행된다.
    - 즉, 그 시점에 가용한 스레드가 없다면 새로 하나를 생성한다.
    - 서버가 아주 무겁다면 CPU 이용이 100% 로 치솟고 도착하는 태스크마다 족족 스레드가 생성되어 더 악화될 것이다.
- 따라서 무거운 서버엔 FixedThreadPool 을 선택하거나 모든 것을 통제 가능한 ThreadPoolExecutor 직접 사용이 낫다.

## 스레드를 직접 다루는 것은 일반적으로 피해라
- 스레드를 직접 다루면 Thread 가 작업 단위와 수행 메커니즘 역할을 모두 수행하게 된다.
    - 수행 메커니즘? 특정 작업 단위(task)를 어떤 스레드가 수행할 것이냐이다.
    - 즉, 직접 다루면 스레드를 관리하는 데 복잡하고 위험이 존재한다.
- 반면 실행자 프레임워크에선 작업 단위와 수행 메커니즘이 분리된다.
    - Executor 가 알아서 태스크를 스레드에 할당하고 생성하고 관리해준다.

## 작업 단위, 태스크(Task)
- 작업 단위를 나타내는 핵심 추상 개념을 task 라고 한다.
- 태스크의 두 종류
    - `Runnable` 과 `Callable`
    - Callable 은 Runnable 과 비슷하지만 차이점은 값을 반환하고 임의의 예외를 던질 수 있다는 점이다.
- 실행자 서비스는 이러한 task 를 수행하는 일반적인 메커니즘이다.
- task 수행을 실행자 서비스에 맡기면 원하는 task 수행 정책을 선택하고, 생각이 바뀌면 손쉽게 변경이 가능하다.
- **핵심은 실행자 프레임워크가 대신 작업 수행을 담당해준다는 것이다.**

### 포크-조인(fork-join) 태스크
- java 7이 되면서 실행자 프레임워크는 fork-join 태스크를 지원하도록 화장되었다.
- 포크 조인 태스크는 포크-조인 풀이라는 특별한 실행자 서비스가 병렬로 작업을 처리해준다.
- 포크 조인 태스크, 즉 ForkJoinTask 의 인스턴스는 작은 하위 태스크로 나뉜다.
- 이러한 태스크는 ForkJoinPool 을 구성하는 스레드들이 이 태스크를 처리하며, 일을 먼저 끝낸 스레드는 다른 스레드의 남은 태스크를 가져와 대신 처리할 수 있다.
- 이렇게 하여 모든 스레드가 바쁘게 움직여 CPU를 최대한 활용하면서 높은 처리량과 낮은 지연시간을 달성한다.
- 포크-조인 태스크를 실제 잘 작성하는 것은 어렵지만, ForkJoinPool 의 병렬 스트림을 이용하면 적은 노력으로 쉽게 사용할 수 있다. (물론 포크-조인에 적합한 형태의 작업이어야 한다...)

## 요약 정리
- 스레드를 직접 다루지 말고 실행자 프레임워크(Executor)를 사용하자.
- 실행자 프레임워크는 작업 단위(task)와 수행 메커니즘을 분리해주어 알아서 작업 수행을 담당해준다는 이점이 있다.

