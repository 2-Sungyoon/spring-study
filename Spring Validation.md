# 📅 [week 1] 주제: Spring Validation

## 🎯 학습 목표
- 파라미터 검증 방식에 대해 학습한다.

---

## 📝 주요 개념 정리
## 1. Bean Validation 개요

Spring은 Jakarta Bean Validation(JSR-380)을 기반으로 입력값 검증을 수행한다.

검증은 어노테이션 기반으로 선언되며, 컨트롤러 진입 시 자동으로 수행된다.

### 주요 목적

- 잘못된 입력 차단
- 비즈니스 로직 진입 전 데이터 검증
- 일관된 오류 처리 구조 제공

---

## 2. `@Valid`의 역할

`@Valid`는 **객체 단위 검증을 트리거**하는 어노테이션이다.

```java
@PostMapping
public ResponseEntity<?> createUser(@Valid @RequestBody UserRequest request) {
    return ResponseEntity.ok().build();
}
```

### DTO 정의

```java
public class UserRequest {

    @NotBlank
    private String name;

    @Email
    private String email;
}
```

### 동작 과정

1. HTTP 요청 → DTO 바인딩
2. `@Valid`가 DTO 검증 수행
3. 제약 조건 위반 시 예외 발생
4. 컨트롤러 메서드 실행되지 않음

---

## 3. 단일 파라미터 검증 방법

`@Valid`는 객체 검증용이므로 단일 값에는 적용되지 않는다.

### 잘못된 사용

```java
public void test(@Valid @RequestParam String name)
```

### 올바른 사용

단일 값은 제약 어노테이션을 직접 적용한다.

```java
@RestController
@Validated
public class UserController {

    @GetMapping
    public void test(
            @RequestParam
            @NotBlank
            @Size(max = 10)
            String name
    ) { }
}
```

---

## 4. `@Validated`의 역할

메서드 파라미터 검증을 활성화하려면 컨트롤러에 `@Validated`가 필요하다.

```java
@RestController
@Validated
public class UserController { }
```

### 기능

- `@RequestParam`, `@PathVariable` 검증 활성화
- 그룹 검증 지원

---

## 5. 검증 위치별 동작 차이

| 입력 위치 | 검증 방법 | 발생 예외 |
| --- | --- | --- |
| RequestBody (DTO) | `@Valid` | MethodArgumentNotValidException |
| RequestParam | 제약 어노테이션 + `@Validated` | ConstraintViolationException |
| PathVariable | 제약 어노테이션 + `@Validated` | ConstraintViolationException |
| ModelAttribute | `@Valid` | BindException |

---

## 6. 중첩 객체 검증

객체 내부 객체 검증은 필드에 `@Valid`를 추가해야 한다.

```java
public class OrderRequest {

    @Valid
    private Address address;
}
```

---

## 7. 주요 제약 어노테이션

### 문자열 검증

- `@NotBlank`
- `@Size`
- `@Pattern`
- `@Email`

### 숫자 검증

- `@Positive`
- `@Min`
- `@Max`

### null 관련

- `@NotNull`
- `@Null`

---

## 8. 검증 실패 시 발생 예외

### 1) RequestBody 검증 실패

```java
MethodArgumentNotValidException
```

### 2) 파라미터 검증 실패

```java
ConstraintViolationException
```

### 3) 바인딩 실패

```java
BindException
```

---

## 9. 글로벌 예외 처리

예외에 대한 처리를 개발자가 명시적으로 처리하지 않을 경우, 스프링부트에서 제공하는 기본 오류 응답을 사용하게 된다. 전역 예외 처리기를 통해 해당 예외를 명시적으로 처리할 수 있다.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidException(
            MethodArgumentNotValidException e) {

        String message = e.getBindingResult()
                .getFieldErrors()
                .get(0)
                .getDefaultMessage();

        return ResponseEntity.badRequest()
                .body(new ErrorResponse("VALIDATION_ERROR", message));
    }
}
```

---

## 10. DTO를 사용하는 것이 권장되는 경우

DTO 사용이 적합한 상황:

- 여러 필드 검증이 필요한 경우
- 중첩 객체 검증이 필요한 경우
- 복잡한 입력 구조
- 확장 가능성이 있는 요청 구조

---
