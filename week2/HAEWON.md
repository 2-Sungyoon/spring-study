# 📅 [week 2] 주제: Spring AOP & Proxy

## 🎯 학습 목표
- Spring AOP의 프록시 기반 동작 원리 이해

- self-invocation 문제와 Bean 분리를 통한 해결 방식 학습

- IoC 컨테이너에서 Bean 등록과 AOP 적용 관계 이해

- AOP 로깅을 통해 공통 관심사 분리 설계 경험

---

## 📝 주요 개념 정리
### 1. 프록시
-
**`프록시란 소프트웨어 패턴의 한 종류로, 어떤 객체에 대한 대리자 또는 대체자 역할을 하는 객체이다.`** 

사용 이유 : 

- 비즈니스 로직과 공통 관심사를 분리하기 위해
    - 여기서 공통 관심사란 - 핵심 로직과는 직접적인 관련이 없지만, 반복적으로 필요한 기능들을 말함
    - ex) log, 트랜잭션 처리, caching, 보안 등등

**Spring AOP 중요 용어 정리**

| **용어** | **뜻** |
| --- | --- |
| **Aspect** | 공통 기능을 모듈화한 것(하나로 묶은 것) |
| **Join Point** | AOP가 적용될 수 있는 시점 |
| **Advice** | aspect 안에 들어있는 실제 동작 코드 |
| **Pointcut** | Join Point 중에서 어디에 적용할지를 고르는 조건 |
| **Target** | 실제 비즈니스 객체(프록시 x), Aspect가 적용될 대상 |
| **Weaving** | aspect를 target에 적용하는 과정 |

**→ `Spring AOP는 프록시를 통해 동작하며, 메서드 호출을 가로채 부가 기능을 실행하는 방식`**

**→ `즉, 스프링에서 AOP가 적용되려면 프록시를 반드시 통과하는 호출이어야 함`** 

**→ `proxy는 객체 외부에서 내부로 들어오는 호출만 가로챌 수 있음`**

### 2. Spring self invocation
Spring AOP 기반 기능(@Transactional, @Cachable등) 은 같은 클래스 내부에서 메서드를 호출하면 동작하지 않을 수 있다. 

**초기 코드**

```java
@Serivce
public class ProductService { 
	private final ProductReposiotry prodectRepository;
	
	public int getProductPrice(Long productId) { 
		Product product = productRepository.findById(productId).orElseThrow();
		return product.getPrice();
	}
}
```

**문제 인식 :** 

- getProductPrice()가 매 요청마다 DB 조회
- 같은 상품인데도 계속 DB 탐

**1차 리팩토링**

```java
@Service
public class ProductService { 
	private final ProductRepository productRespository;
	
	public int getProdectPrice(Long productId) { 
		Product product = findCachedProduct(productId);
		return product.getPrice();
	}
	
	@Cacheable("product")
	private Product findCachedProduct(Long productId) { 
		return productRepository.findById(productId).orElseThorw();
	}
}
```

기대 : 

- @Cacheable 붙였으니 같은 productId일 경우 DB 조회가 아니라 캐시를 쓰겠지?

![image.png](attachment:512afc1b-d63a-4713-ab07-0e30817104eb:image.png)

실제 : 

- 요청 올때마다 DB 계속 조회됨
- why?
    - Product product = findCachedProduct(productId);는 this.findCachedProduct(productId)와 동일
    - 같은 클래스 내 메서드 호출은 프록시를 거치지 않음(self invocation 문제)
    
    **`→ AOP 대상 메서드를 다른 Bean으로 분리해야한다.`**
    
    ![image.png](attachment:f557e995-60db-4d3b-b032-6eb696964553:image.png)
    

**최종 해결**

```java
@Service
public class ProductQueryService {

    private final ProductRepository productRepository;

    @Cacheable("product")
    public Product findProduct(Long productId) {
        return productRepository.findById(productId)
                .orElseThrow();
    }
}
```

```java
@Service
public class ProductService {

    private final ProductQueryService productQueryService;

    public int getProductPrice(Long productId) {
        Product product = productQueryService.findProduct(productId);
        return product.getPrice();
    }
}
```

→ ProductQueryService(AOP 대상 메서드) & ProductService로 분리

**한줄 요약**

> Spring의 프록시 기반 AOP는 같은 클래스 내부 호출(self-invocation)에서는 적용되지 않는다.
>

### 3. AOP Logging
사용하는 이유 : 

- 비즈니스 로직과 공통 관심사를 분리하기 위해
- 하나의 코드로 관리하기 때문에 유지보수와 수정, 확장이 편리함
- 코드의 재사용성 증가

**사용하는 방법**

1. 의존성 확인
- AOP Starter 의존성 추가

```java
implementation("org.springframework.boot:spring-boot-starter-aop")
```

1. 로깅 범위 정하기
- 패키지 / 어노테이션 / 커스텀 어노테이션 기준
- 주로 패키지 많이 잡음

1. Aspect 클래스 만들기
- @Aspect + @Component 붙이기
- 주로 @Around Advice를 많이 씀
    - 메서드 실행 전/후/예외까지 전체 흐름을 제어할 수 있기 때문

```java
@Aspect
@Component
@Slf4j
public class LoggingAspect {

    // service 패키지 아래 모든 메서드
    @Around("execution(* com.example..service..*(..))")
    public Object logExecution(ProceedingJoinPoint joinPoint) throws Throwable {
        
        Object result = joinPoint.proceed();
				return result;
    }
}
```

> @Aspect는 AOP 역할을 지정하는 어노테이션이며,
> 
> 
> 실제 동작을 위해서는 `@Component` 등을 통해 Bean 등록이 필요하다.
>
---

## 💻 코드 실습 및 분석
```java

```

---

## 💡 회고 및 질의응답
<aside>

전에는 AOP를 어노테이션만 붙이면 되는 기능으로 생각했는데,

이번주 스터디를 통해 Spring AOP가 프록시 기반으로 동작하고 self-invocation 문제가 코드 구조의 문제라는 것을 알게 되었다.

또한, 이때까지는 직접 하나하나 로그를 필요한 위치에 찍었는데, 다음 프로젝트에서는 AOP logging을 활용해보고 싶다.

</aside>
  
