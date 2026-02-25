# 📅 [week 5] 주제: Thread & Executor

## 🎯 학습 목표
- Executor와 ThreadPoolExecutor의 구조와 동작 원리를 이해한다.

- WorkQueue 전략과 포화 정책의 차이를 구분할 수 있다.

- Spring @Async가 Java Executor 기반 위에서 동작함을 설명할 수 있다.

---

## 📝 주요 개념 정리
### 1. 소제목
### 1. Thread & Task

**`Thread란, 프로세스 안에서 실제로 작업을 수행하는 가장 작은 실행 단위이다.`**

- Task = 논리적인 단위
- Thread = Task를 실행하는 기술적인 수단

Spring MVC 환경에서는 기본적으로 `요청 1개 = 스레드 1개`

**기존 방식**

```java
 new Thread(() -> {
    System.out.println("작업 실행");
}).start();
```

문제점 : 

- 요청마다 스레드 생성
- 생성 비용 ⬆️
- 관리 어려움
- 스레드 수 제어 불가능

→ 서버에서 직접 new Thread()를 쓰는 것은 매우 비효율적

### 2. Executor

**`Executor는 Task 실행을 추상화한 인터페이스이다.`**

```java
public interface Executor {
    void execute(Runnable command);
}
```

- 작업 실행을 추상화한 인터페이스
- 스레드를 직접 다루지 않게 해줌
- 실행 정책을 내부에 위임

→ 스레드를 직접 다루지 말고, 작업 실행을 위임하자

### **3. ExecutorService**

**`Executor를 확장한 인터페이스로, 라이프사이클 관리 + 비동기 결과 처리가 가능하다.`**

```java
public interface ExecutorService extends Executor {

    void shutdown();
    List<Runnable> shutdownNow();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit);
}
```

필요한 이유 : 

- Executor는 작업을 비동기적으로 실행하기 때문에, 제출된 작업이 언제 끝나는지 모름
    - 서버 종료 시 남아 있는 작업 정리
    - 현재 실행 중인 작업 처리 방식 결정
    - 대기 중인 작업의 처리 여부 결정
    - 안전하고 예측 가능한 종료 보장

→ 비동기 작업은 단순 실행뿐만 아니라, 종료 시점까지 포함한 생명주기 관리 필요

→ shutdown(), awaitTerminiation() 같은 메서드를 통해 비동기 작업의 전체 생명주기 관리하도록 설계

### 4. ThreadPoolExecutor

```java
public ThreadPoolExecutor(
    int corePoolSize,
    int maximumPoolSize,
    long keepAliveTime,
    TimeUnit unit,
    BlockingQueue<Runnable> workQueue,
    RejectedExecutionHandler handler
)
```

- corePoolSize : 항상 유지되는 최소 스레드 수
- maximumPoolSize : 큐가 꽉 찼을 때 늘어날 수 있는 최대 스레드 수
- keepAliveTime : corePoolSize를 초과해 생성된 스레드가 일을 하지 않고 기다리는 상태일 때, 얼마 동안 유지할 것인지 정하는 시간
- unit : keepAliveTime의 시간 단위
- workQueue : 실행 대기 중인 Task를 저장하는 큐
- handler : 큐가 가득 찼을 때 어떻게 처리할지

**4-1. WorkQueue**

1. unbounded queue (무한 큐)
- 기본 newFixedThreadPool()이 사용
- 요청이 몰리면 큐가 계속 증가
- 메모리 위험 존재

1. bounded queue (제한 큐)
- 일정 크기 이상이면 작업 거부
- 서버 안정성 확보 가능

→ 실무에서는 bounded 큐 전략이 더 안전함

**4-2. 포화 정책(Saturation policies)**

스레드 풀과 큐가 모두 가득 찼을 때, 어떻게 할 것인가

- AbortPolicy(기본)
    - 작업 대기열 가득 차면 RejectedExecutionException 발생
- CallerRunsPolicy
    - 일종의 스로틀링(throttling)
    - 작업을 제출한 스레드가 직접 해당 작업을 실행
        - 비동기 풀에 던지지 X
    - 작업을 제출한 스레드가 해당 작업을 처리하는 동안 다음 요청 못 받음
    - 요청 처리 속도가 느려지고, 시스템 전체의 유입 속도가 조절됨
    - 작업 무시 X, 예외 발생 X
- DiscardPolicy
    - 새 작업 무시
- DiscardOldestPolicy
    - 가장 오래된 작업 제거 후, 새 작업 추가

### 5. Spring과 연결

```json
@Async 붙은 메서드 호출
        ↓
Spring 프록시가 가로챔
        ↓
TaskExecutor에게 작업 전달
        ↓
ThreadPoolTaskExecutor가 작업을 내부 풀에 위임
        ↓
ThreadPoolExecutor가 실제 스레드에서 실행
```

---

## 💻 코드 실습 및 분석
```java

```

---

## 💡 회고 및 질의응답
처음에는 queueCapacity를 설정하면 모든 경우에 큐 전략이 동일하게 결정되는 줄 알았다.

ThreadPoolExecutor에서 newFixedThreadPool(기본이 unboundedqueue)을 기본으로 쓰는데 LinkedBlockingQueue를 사용한다고 했고, ThreadPoolTaskExecutor에서 queueCapacity를 통해 값을 설정하면 LinkedBlokingQueue를 쓴다고 해서, 그럼 항상 같은 큐를 쓰는게 아닌가 싶어서 헷갈렸다.

- Spring(ThreadPoolTaskExecutor)
    
    → `queueCapacity` 값에 따라
    
    0보다 클 경우 LinkedBlockingQueue(제한), 
    
    0이면 SynchronousQueue가 내부적으로 생성된다.
    
- Java(ThreadPoolExecutor)
    
    → queueCapacity 개념은 없고,
    
    생성자에 어떤 `BlockingQueue`를 넘기느냐에 따라 전략이 결정된다.
    

**+ 왜 기본 Executor(ex : newFixedThreadPool 같은 애들)을 사용하는 것이 좋지 않나?**

→ 내부 설정이 명시적으로 보여지지 않고, 튜닝할 수 없기 때문에 각 환경에 맞게 설계할 수가 없음
