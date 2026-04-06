# 📅 [week N] 주제: CORS Configuration

## 🎯 학습 목표
- Spring Boot 프로젝트에서 CORS 관련 설정 방식에 대해 학습한다.

---

## 📝 주요 개념 정리
## 1. CORS와 WebConfig/SecurityConfig

### CORS (Cross-Origin Resource Sharing)

CORS는 브라우저의 보안 정책(Same-Origin Policy)을 완화하기 위해,

서버가 특정 출처(origin)의 요청을 허용할지 명시적으로 결정할 수 있도록 하는 메커니즘이다.

브라우저는 다른 출처에서 요청이 발생할 경우,

서버의 CORS 응답 헤더를 확인하여 해당 요청을 허용하거나 차단한다.

### **WebConfig (`WebMvcConfigurer`)**

Spring MVC의 동작 방식을 구성하는 설정으로, CORS, 인터셉터, 리소스 핸들링 등 웹 요청 처리 규칙(MVC 레벨)을 정의한다.

### **SecurityConfig (`SecurityFilterChain`)**

Spring Security의 인증 및 인가 정책을 구성하는 설정으로, 요청에 대한 접근 제어, 필터 체인, 세션 정책 등을 정의하며 요청이 MVC에 도달하기 전 단계(보안 레벨)에서 동작한다.

Spring Security 없이 Spring MVC만 사용하는 구조에서는 `WebConfig`를 통해 CORS를 허용하면 정상적으로 동작한다. Spring Framework는 MVC 차원에서 전역 CORS 설정을 지원하며, `WebMvcConfigurer`의 `addCorsMappings()`를 통해 이를 구성할 수 있다.

- `WebConfig`에서 `addCorsMappings()` 등록
- 인증/인가가 없음
- 요청이 단순히 MVC까지 도달

하지만 JWT 도입과 함께 Spring Security가 활성화되면 요청 흐름이 달라진다. 

- 요청이 먼저 Security Filter Chain에 도달
- 그 이후에 DispatcherServlet → MVC
- preflight `OPTIONS` 요청도 먼저 Security를 거침

즉, Spring Security가 활성화된 환경에서는 요청이 MVC까지 도달하기 전에 먼저 Security Filter Chain을 통과하므로, CORS 역시 MVC가 아니라 Security 단계에서 함께 처리되어야 한다. 특히 preflight `OPTIONS` 요청은 인증 정보 없이 들어오는 경우가 많아, Security가 이를 먼저 적절히 처리하지 않으면 DispatcherServlet이나 Controller까지 전달되기 전에 차단될 수 있다. 따라서 Spring Security를 사용하는 애플리케이션에서는 CORS 정책을 Security와 통합하여 구성해야 한다.

---

## 2. WebConfig : Spring MVC 중심 CORS 설정

Spring Security를 사용하지 않는 시점에는 다음처럼 작성한다.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOriginPatterns("http://localhost:*")
                .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```

이 방식의 의미는 다음과 같다.

- MVC가 전역 CORS 정책을 가진다
- 브라우저의 교차 출처 요청을 MVC가 허용한다
- 요청이 정상적으로 Controller까지 들어간다

Spring MVC는 전역 CORS 구성과 세밀한 CORS 제어를 지원하며, Spring Security를 사용하지 않는 경우 해당 설정만으로도 동작한다.

---

## 3. 왜 Spring Security가 들어오면 문제가 생기는가

Spring Security가 활성화되면 요청 흐름은 다음처럼 바뀐다.

브라우저 요청 → Security Filter Chain → DispatcherServlet → Controller → 응답

즉, preflight `OPTIONS` 요청도 먼저 Security를 통과한다.

Spring Security 공식 문서는 **CORS가 Security보다 먼저 처리되어야 한다**고 설명한다. 그 이유는 preflight 요청에는 일반적으로 쿠키 같은 인증 정보가 없어서, Security가 이를 인증되지 않은 요청으로 오해할 수 있기 때문이다. 또한 Spring Security는 `CorsConfigurationSource`를 사용해 CORS를 통합할 수 있다고 안내한다.

프로젝트에서 겪을 수 있는 의문점

- “WebConfig에 CORS 넣어놨는데 왜 안 되지?”
- “JWT는 Authorization 헤더만 쓰니까 CORS랑 상관없는 거 아닌가?”
- “SecurityConfig에 `.cors()`만 켜면 끝 아닌가?”

의문에 대한 답

- MVC에만 CORS를 두면 Security 단계에서 먼저 막힐 수 있다
- JWT를 Authorization 헤더로 보내더라도, 브라우저 기반 cross-origin 요청 자체가 CORS 검사 대상이다
- Security에서 `.cors(...)`를 활성화하더라도, 참조할 CORS 정책 원천이 분명해야 한다

---

## 4. SecurityConfig : CorsConfigurationSource를 Bean으로 두고 Security에서 사용

Spring Security 공식 문서는 `CorsConfigurationSource`를 제공해서 Security와 통합하는 방식을 예시로 제시한다. 특히 `UrlBasedCorsConfigurationSource`를 사용하면 각 경로에 대한 CORS 정책을 등록할 수 있고, Security는 이를 바탕으로 CORS를 처리할 수 있다.

여기서 중요한 점은, 이번 정리에서는 **생성자 필드 주입이 아니라 `@Bean` 메서드 파라미터 주입 방식**을 기준으로 예시를 잡는다는 것이다. Spring의 DI는 생성자 인자뿐 아니라 **팩토리 메서드 인자**, 즉 `@Bean` 메서드 파라미터를 통해서도 이루어진다.

### 4-1. CORS 설정 Bean

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();

        configuration.setAllowedOriginPatterns(List.of("http://localhost:*"));
        configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"));
        configuration.setAllowedHeaders(List.of("*"));
        configuration.setExposedHeaders(List.of("Authorization"));
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);

        return source;
    }
}
```

클래스의 역할

- CORS 정책 자체를 정의한다
- 어떤 origin, method, header를 허용할지 결정한다
- Security가 재사용할 수 있는 Bean을 제공한다

---

### 4-2. SecurityConfig에서 CORS 정책 사용

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(
            HttpSecurity http,
            CorsConfigurationSource corsConfigurationSource,
            JwtAuthenticationFilter jwtAuthenticationFilter
    ) throws Exception {

        http
                .cors(cors -> cors.configurationSource(corsConfigurationSource))
                .csrf(csrf -> csrf.disable())
                .sessionManagement(session ->
                        session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                )
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/public/**").permitAll()
                        .anyRequest().authenticated()
                )
                .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

이 구조의 장점

- CORS 정책은 `CorsConfigurationSource`가 담당한다
- Security는 그 Bean을 파라미터로 주입받아 사용한다
- 의존관계가 메서드 시그니처에서 드러난다
- 설정 클래스 자체를 무겁게 만들지 않고, 필요한 Bean만 명시적으로 연결할 수 있다

Spring Framework는 `@Configuration` 클래스와 `@Bean`을 통한 Java 기반 설정을 지원하며, `@Bean` 메서드 간 의존관계 또는 다른 Bean 참조를 메서드 인자로 선언할 수 있다. Spring Security도 `SecurityFilterChain`을 `@Bean`으로 등록하는 구성을 기본 방식으로 설명한다.

---

## 5. SecurityConfig : MVC CORS 설정을 유지하고 Security에서 .cors()만 활성화

Spring Security 문서는 Spring MVC가 있고 별도의 `CorsConfigurationSource`가 없으면, Security가 MVC의 CORS 구성을 사용할 수 있다고 설명한다. 즉, 다음 구조도 가능하다.

### MVC 설정

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOriginPatterns("http://localhost:*")
                .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```

### Security 설정

```java
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
                .cors(Customizer.withDefaults())
                .csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/public/**").permitAll()
                        .anyRequest().authenticated()
                );

        return http.build();
    }
}
```

---

## 💡 회고 및 질의응답
- 해커톤에서는 구글 로그인을 사용하지 않았기에, WebConfig 설정만으로 CORS 관련 문제를 해결할 수 있었다. 소셜로그인을 도입하면서 Spring Security를 활성화한 다른 팀에서 CORS 관련 문제를 겪을 때,
우리 팀의 설정을 공유했지만 도움이 되지 못했던 기억이 있다. 겨울잠에서는 JWT를 사용했는데 왜 문제가 없었는지 코드를 살펴보니 클래스 위치는 마음에 안들지만 CorsConfigurationSource를 어떻게든 사용하고 있었다.
프로젝트에서 제일 이해하기 힘든 부분이었는데 고민 해결!
