# 📅 [week 1] 주제: Spring MVC의 요청 처리 구조와 실행 흐름

## 🎯 학습 목표
- 웹 서버와 WAS의 차이 이해
- 서블릿과 Spring MVC 패턴 이해

---

## 📝 주요 개념 정리
# 웹 서버

### 정의

> HTTP 요청을 받아 정적인 리소스(HTML, CSS, JS, 이미지 등)를 제공하는 서버
> 

### 주요 역할

- 클라이언트의 HTTP 요청 수신
- 정적 파일 응답
- 리버스 프록시 역할 (요청 전달)
- SSL/TLS 처리
- 로드 밸런싱

### 특징

- 요청에 대한 **비즈니스 로직 실행은 하지 않음음**
- 빠르고 가벼움
- 대량의 정적 리소스 처리에 최적화

### 대표적인 예

- Nginx
- Apache HTTP Server

# WAS (Web Application Server)

### 정의

> 웹 애플리케이션을 실행하기 위한 서버로,
HTTP 요청을 받아 동적인 비즈니스 로직을 수행하고 응답을 생성하는 서버
> 

### 주요 역할

- HTTP 요청 처리
- 비즈니스 로직 실행
- 데이터베이스 연동
- 트랜잭션 관리
- **서블릿 컨테이너 제공**

### 특징

- 애플리케이션 실행 환경 제공
- 멀티스레드 기반 요청 처리
- 웹 서버보다 무겁지만 기능이 많음

### 대표적인 예

- Apache Tomcat
- JBoss
- WebLogic

---

# 서블릿 (Servlet)

### 정의

> HTTP 요청을 Java 코드로 처리하기 위한 표준 인터페이스이며,
WAS가 호출해주는 서버 측 컴포넌트
> 

### 주요 역할

- HTTP 요청/응답 처리 로직 제공
- `HttpServletRequest`, `HttpServletResponse` 사용
- WAS와 애플리케이션 코드 사이의 연결 지점

### 특징

- 개발자가 직접 호출하지 않음
- WAS가 생명주기 관리 (`init → service → destroy`)
- 보통 싱글 인스턴스 + 멀티스레드 구조
- HTTP 패킷을 직접 다루지 않음 (WAS가 파싱)

### 대표적인 예

- Spring MVC의 `DispatcherServlet`

# Spring MVC

## 정의

> 서블릿(DispatcherServlet)을 기반으로
HTTP 요청을 Controller–Service–View 구조로 처리하는
Spring의 웹 애플리케이션 프레임워크
> 

---

## Spring MVC의 핵심 개념

### 1. Front Controller 패턴

- 모든 HTTP 요청을 **하나의 서블릿**이 먼저 받음
- 그 서블릿이 요청을 적절한 컨트롤러로 분배

👉 Spring MVC에서 이 역할을 하는 것이 **DispatcherServlet**이다.

---

### 2. 요청 처리 책임 분리 (MVC)

| 역할 | 설명 |
| --- | --- |
| Model | 비즈니스 데이터 |
| View | 화면 렌더링 (HTML 등) |
| Controller | 요청 처리 및 흐름 제어 |


_💡REST API에서는 View 없이 **Model → JSON**으로 바로 응답_



---

## Spring MVC 구성 요소

| 구성 요소 | 역할 |
| --- | --- |
| DispatcherServlet | 모든 요청의 진입점 (서블릿) |
| HandlerMapping | URL → Controller 메서드 매핑 |
| HandlerAdapter | Controller 실행 방식 결정 |
| Controller | 비즈니스 요청 처리 |
| ViewResolver | View 이름 → 실제 View |
| HttpMessageConverter | 객체 ↔ JSON / Text 변환 |

---

## 요청 처리 흐름

<img width="709" height="347" alt="image (1)" src="https://github.com/user-attachments/assets/de2e67a8-d9e1-4533-a94e-ac8797dbf25a" />


1. 핸들러 조회 : 핸들러 매핑을 통해 URL에 매핑된 핸들러(컨트롤러)를 조회한다.
2. 핸들러 어댑터 조회 : 핸들러를 실행할 수 있는 핸들러 어댑터를 조회한다.
3. 핸들러 어댑터 실행 : 핸들러 어댑터를 실행한다.
4. 핸들러 실행 : 핸들러 어댑터가 실제 핸들러를 실행한다.
5. ModelAndView 반환 : 핸들러 어댑터는 핸들러가 반환하는 정보를 ModelAndView로 변환하여 반환한다.
6. viewResolver 호출 : 뷰 리졸버를 찾고 실행한다.
7. View 반환 : 뷰 리졸버는 뷰의 논리 이름을 물리 이름으로 바꾸고, 렌더링 역할을 담당하는 뷰 객체를 반환한다.
8. 뷰 렌더링 : 뷰를 통해 뷰를 렌더링한다.

---

## Spring MVC에서 중요한 포인트

### Controller는 서블릿이 아님

- HTTP 요청을 **직접 받지 않음**
- DispatcherServlet이 호출해 줌

### 서블릿은 하나, 컨트롤러는 여러 개

- 서블릿: DispatcherServlet 1개
- Controller: 기능별 여러 개

### 서블릿 API를 숨겨줌

- `HttpServletRequest`를 직접 다루지 않아도 된다.
- 어노테이션 기반 (`@GetMapping`, `@RequestBody` 등)

---

## Spring Boot와의 관계

> Spring MVC는 Spring Boot 안에서 동작하는 웹 프레임워크다.
> 
- Spring Boot
    - 내장 WAS 제공
    - 자동 설정
    - 실행 환경
- Spring MVC
    - HTTP 요청 처리 방식 제공

💡 **Spring MVC는 독립 실행 앱이 아니다**

---

## 💻 코드 실습 및 분석
## 템플릿 vs REST API 분기

### 템플릿 기반

```java
@Controller
public class HomeController {

    @GetMapping("/home")
    public String home() {
        return "home";
    }
}

```

- ViewResolver 사용
- HTML 응답

### REST API

```java
@RestController
public class UserController {

    @GetMapping("/users/me")
    public UserResponse me() {
        return new UserResponse("Tom", 20);
    }
}

```

- HttpMessageConverter 사용
- JSON 응답
- @Restcontroller는 @Controller와 @ResponseBody를 포함

---

## 💡 회고 및 질의응답
- 클라이언트의 요청이 처리되어 반환되는 흐름을 정리할 수 있었고, 템플릿을 사용하는 경우와 json을 통한 HTTP응답 방식의 차이를 이해할 수 있었다.
- 웹 서비스를 구현할 때, 사용 이유를 정확하게 모르는 채 @Restcontroller를 사용하곤 했었다. 이때 @ResponseBody의 역할과, 데이터를 어떤 방식으로 클라이언트에게 전달하는지 알 수 있었다.

## 참고자료
> [스프링 MVC 1편 - 백엔드 웹 개발 핵심 기술(김영한)](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-mvc-1/dashboard)
