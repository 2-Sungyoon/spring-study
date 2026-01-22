# 📅 [week 1] 주제: Spring Core & Bean Management 정리

## 🎯 학습 목표
- Spring core와 IoC 컨테이너 이해
- Bean의 다양한 등록 방식 학습
- Bean의 등록 방식을 상황에 맞게 선택

---

## 📝 주요 개념 정리
### 1. 스프링 생태계
- **용어정리**

| **용어** | **뜻** | 실무 관점 |
| --- | --- | --- |
| **1. spring core** | 스프링의 핵심 기능으로, IoC와 DI를 제공하는 가장 기초적인 모듈 |  |
| **1-1. spring context** | spring core를 기반으로, 앱의 Bean을 생성.관리하는 IoC 컨테이너를 제공하는 모듈 |  |
| **1-2. spring aspects** | AOP를 지원하는 모듈로, 트랜잭션, 로깅, 보안 같은 공통 관심사를 로직과 분리할 수 있게 해줌 | * AOP : 스프링 관점 지향 프로그래밍 |
| **2. MVC** | 웹 앱에서 HTTP 요청을 처리하기 위한 모듈로, Model-View-Controller 구조를 기반으로 요청과 응답을 분리함 |  |
| **3.spring data access** | DB 접근을 쉽게 하기 위한 모듈로, JDBC, JPA, 트랜잭션 관리 등을 통해 영속성 계층 구현을 지원한다. |  |
| **4. spring testing** | 스프링 환경에서 단위 테스트 및 통합 테스트를 쉽게 작성할 수 있도록 컨텍스트 로딩, Mock 객체 등을 지원하는 테스트 모듈 | * 컨텍스트 로딩 : 스프링이 앱을 시작할때 Bean들을 전부 준비하는 과정 |
| **5. Bean** | 스프링 컨테이너가 관리하는 객체(인스턴스) | 의존성 주입, AOP, 트랜잭션 적용 대상 |

### 2. Spring Core
**IoC(Inversion of Control)**
**`IoC란, 객체의 생성과 실행 흐름에 대한 제어권을 애플리케이션 코드가 아닌 스프링 프레임워크가 가지는 구조이다.`**

→ 객체 간 결합도 감소, 테스트 및 확장 용이

method aspecting

- 메서드 애스펙팅
- IoC 컨테이너에 추가된 인스턴스의 동작인 메서드 가로채기

스프링은 MVC를 사용하여 표준 자바 **서블릿 방식**으로 앱을 개발할 수 있게함

- 서블릿 : 브라우저에서 사용자의 요청을 처리해주는 역할
- 현재는 서블릿 직접구현 x, @DispatchServlet 기반으로 @GetMapping 같은 애들 씀

| **용어** | **뜻** |
| --- | --- |
| ORM | 자바 객체를 SQL 테이블과 자동 연결해주는 프레임워크 |
| JPA | ORM 사용법에 대한 규칙 |
| Hibernate | JPA 기반으로 실제구현하는 것(ORM 엔진) |

**설정 파일**
- .properties 파일
    - 애플리케이션 동작에 필요한 설정값을 코드 밖에서 관리하기 위한 파일
    - .yml 파일과의 차이점 :
        - 들여쓰기 정도의 차이
        - 둘 중에 하나만 사용하면 됨
- .env 파일
    - 환경변수를 파일로 정리해둔 것
    - 매번 export하기 귀찮으니까
    - 표준이 아니라, Spring이 자동으로 안 읽음
- .docker-compose.yml 파일
    - 여러 개의 컨테이너를 한 번에 정의해서 실행하는 설정 파일

<aside>

**그렇다면 실전에서는 어떤 조합으로 사용할까?**

→ **`.env + docker-compose.yml + applicatoin.yml(prod, local 분리 가능)`**

</aside>

### spring context에 새로운 Bean 추가

**1. @Bean 애너테이션 사용(설정 클래스에서 수동 등록)**

- 개념
    - 설정 클래스에서 메서드가 반환하는 객체를 빈으로 등록
    - 보통 @Configuration 클래스 안에 @Bean 을 붙여서 사용
- 어떻게 등록되나
    - 컨테이너가 @Configuration 클래스를 읽음
    - @Bean 메서드를 실행해서 객체를 만들고, 그 반환값을 빈으로 등록
- 장점
    - 외부 라이브러리도 빈으로 등록가능
    - 세밀한 커스터마이징 가능
    - @Bean 메서드 호출 결과 싱글톤 보장(프록시)
- 단점
    - 내가 만든 로직 클래스에 쓰기엔 귀찮고 과함

**2. 스테레오타입 애너테이션 사용(@Component 계열, 자동 등록)**

- 개념
    - 클래스에 애너테이션 붙여서 스프링이 자동 스캔으로 빈 등록
    - @Component, @Service, @Repository, @Controller
- 어떻게 등록되나
    - @ComponentScan이 지정 패키지를 훑음
    - @Component 계열이 붙은 클래스를 발견하면 스프링이 객체를 만들고 빈으로 등록
- 장점
    - 실무에서 자주 사용
    - 코드에서 역할이 명확히 드러남
    - 레이어 구조 굿
- 단점
    - 외부 라이브러리 클래스엔 애너테이션 붙일 수 x

**3. 프로그래밍 방식(런타임에 코드로 등록 : registerBean, BeanDefinition 등)**

- 개념
    - 애너테이션 없이, 코드로 빈 등록을 명령하는 방식
    - 동적 등록
- 어떻게 등록되나
    - 개발자가 객체를 만들거나 공급자(supplier)를 준비하고 컨텍스트에 registerBean()으로 등록함
- 장점
    - 실행 중 여러 상황에 따라 빈을 동적으로 등록/교체 가능
    - 플러그인 구조 같은 구성 가능
- 단점
    - 일반 앱 개발시 과함

→  **`1번은 외부 시스템이랑 통신하기 위해(도구),`** 

**`2번은 해당 앱이 실제로 어떻게 동작하는지(우리가 짠 로직),`** 

**`3번은 테스트 / 런타임 교체 / 플러그인 같은 특수한 상황에 사용한다`**

---

## 💻 코드 실습 및 분석
```java

```

---

## 💡 회고 및 질의응답
- 내가 착각한 것 & 알아두면 좋은 것 :
    - 1, 2, 3번 중에 하나의 방식을 쓰는게 아니라, 하나의 프로젝트 안에서도 필요에 따라 섞어 사용할 수 있음
    - @Controller, @Service, @Repository 같은 애들이 모두 @Component의 특수 버전임(스테레오타입)
    - ex)
        
        @Service ∈ @Component
        
        ```java
        @Component
        public @interface Service {
        }
        ```
        
        @Controller ∈ @Component
        
        ```java
        @Component
        public @interface Controller {
        }
        ```
        
        @Repository ∈ @Component
        
        ```java
        @Component
        public @interface Repository {
        }
        ```
        
- 요즘은 RestTemplate(동기)보다 WebClient(비동기) 방식을 더 많이 씀
    - 요청에 대한 응답이 올때까지 다른 작업을 수행하지 못하면 서버 자원을 효율적으로 사용할 수 없음 → 같은 스레드로 여러 요청을 처리하도록 함
    - WebClient에도 .block()을 자주 쓰는데, 이러면 그냥 동기임.
    - 근데 왜 그럼에도 불구하고 WebClient를 사용하냐?
      → 언제든 .block() 제거하고 비동기로 확장 가능
    - 그럼 왜 .block()을 쓰냐?
      → 이미 서버 로직을 동기(Spring MVC)로 짰는데 중간에 비동기 끼워넣으면 전체 구조를 다 바꿔야함 & 외부 API 호출 결과가 바로 필요하기 때문(GPT의 응답을 받아야 다음 로직이 가능하다던가)
