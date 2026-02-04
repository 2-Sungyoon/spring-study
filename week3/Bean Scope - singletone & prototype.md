# 📅 [week 3] 주제: Bean Scope - singletone & prototype

## 🎯 학습 목표
- Spring Bean의 스코프와 생명주기 이해

- Singleton, Prototype 스코프의 차이와 사용 기준 파악

---

## 📝 주요 개념 정리
### 1. Scope
scope :

- 스프링 컨테이너가 그 빈 객체를 언제 만들고, 얼마나 오래, 몇개를 유지하느냐를 정하는 범위

종류 : 

- 싱글톤
- 프로토타입
- 웹스코프
    - Request
    - Session
    - Application 등등..

### 2. 싱글톤
싱글톤 : **`같은 이름의 빈은 하나만 존재한다`**

- 하나의 컨테이너 안에서 같은 빈 이름에 대해 하나의 객체만 존재하는 것
- 아무런 설정안하면 기본으로 싱글톤

**DI(의존성 주입) :** 

- 스프링 컨테이너는 빈을 생성하는 역할 → 빈 사이의 관계를 정해주지는 X
- 빈 사이의 참조 관계는 개발자가 명시해야함
- 이 연결을 스프링이 대신 해주도록 하는 것이 DI

**@Autowired(필드 주입):** 

- 스프링 컨테이너에 있는 빈을 찾아서 주입해달라는 것

```java
@Autowired
private CommentRepository commentRepository;
```

- 우선, **`타입 기준`**으로 빈 탐색
    - 같은 타입 빈이 1개면 자동으로 결정 & 주입
    - 같은 타입 빈이 여러개일 경우, 에러 발생
    - 해결방법 :
        - @Qualifier : 이 이름의 빈을 쓰라고 명시적으로 지정하는 방식
        
        ```java
        @Service
        public class CommentService {
        
            private final CommentRepository repo;
        
            public CommentService(
                @Qualifier("jpaCommentRepository") CommentRepository repo
            ) {
                this.repo = repo;
            }
        }
        ```
        
        - @Primary : 기본값을 지정하는 방식
        
        ```java
        @Primary
        @Repository
        public class JpaCommentRepository implements CommentRepository {}
        ```
        

**생성자 주입 :** 

- 생성자를 통해 의존성을 주입하는 것

```java
@Service
public class CommentService {

    private final CommentRepository commentRepository;

    public CommentService(CommentRepository commentRepository) {
        this.commentRepository = commentRepository;
    }
}
```

- 생성자가 1개일 경우, @Autowired 생략 가능

**Lombok을 통한 생성자 주입 :** 

- RequiredArgsConstructor를 통해 생성자를 자동 생성하는 것

```java
@RequiredArgsConstructor
@Service
public class CommentService {

    private final CommentRepository commentRepository;
}
```

<aside>

**왜 주로 싱글톤(불변)인가?**

- 스프링 애플리케이션은 기본적으로 멀티 스레드 환경에서 동작함
- 싱글톤 빈은 여러 스레드가 동일한 객체(하나의 인스턴스)를 공유함
- race condition(경쟁 상태) 발생 가능
    - 인스턴스를 불변 상태로 만드는 것이 필요함(**`final`**)

→ 그렇기에, **`필드 주입 방식보다 생성자 주입 방식이 권장됨`**

</aside>

### 3. 프로토타입
프로토타입 : **`모든 요청에 대해 별개의 인스턴스 생성`**

- 빈에 대한 참조를 요청받을 때마다 새로운 인스턴스를 생성(공유 X)
- 싱글톤과 달리, 스프링이 직접 객체 인스턴스의 생성까지만 담당하고, 생성 이후에는 더 이상 관여하지 않음
- **`@Scope(BeanDefinition.SCOPE_PROTOTYPE)`** 애너테이션을 통해 명시해야함

**특징 :** 

- 빈을 요청하는 각각의 스레드가 모두 서로 다른 인스턴스를 얻기 때문에 race condition 발생 X
    - 변경 가능한(mutable) 프로토타입 빈을 정의해도 괜찮음
- 더 이상 참조하지 않는 빈 리소스의 소멸 시점은 개발자가 명시해야함(컨테이너가 관리 X)

**`기본은 싱글톤, 예외적으로 프로토타입`**

<aside>

**언제 프로토타입을 사용하면 좋은가?**

- 상태를 가진 빈을 사용하는 경우
- 멀티스레드 환경에서 Thred Safety 보장이 필요한 경우
- 생성 비용이 크고 무거운 객체를 사용하는 경우
    - 필요 시에만 생성, 할당, 자원 해제
</aside>
---

## 💻 코드 실습 및 분석
```java

```

---

## 💡 회고 및 질의응답
**getBean()으로도 빈의 의존 관계 주입 가능한가 ?**

→ @Autowired는 의존 관계를 선언하는 것이고, getBean()은 이미 생성된 빈을 조회해서 가져오는 것

**빈이 하나면 자동 주입된다 ?**

→ 자동이라는게 그냥 알아서 찾아서 연결해준다는게 아님

→ 주입을 해달라고 표시(@Autowired)하면, 스프링이 주입해줄 수 있다는 뜻

**프로토타입 빈의 생명주기가 짧다?**

- 싱글톤의 경우, 빈의 생성부터 소멸까지를 관리함
    - 스프링 컨테이너와 생명 주기가 같음
- 프로토타입의 경우, 요청 시에만 생성이 되고, 그 이후는 컨테이너가 관여 X
    - 스프링 컨테이너 생성 이후 빈이 생성되고, 소멸 시점은 명시되지 않음

**`→ 스프링 컨테이너가 관리하는 생명 주기가 짧다`**
