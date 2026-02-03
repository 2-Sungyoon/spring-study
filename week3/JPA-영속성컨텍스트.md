# 📅 [week 3] 주제: 

## 🎯 학습 목표
- 영속성 컨텍스트의 개념에 대해 이해한다.

---

## 📝 주요 개념 정리
# 영속성 컨텍스트

영속성 컨텍스트(Persistence Context)는 엔티티를 영구 저장하는 환경으로, **객체 중심의 데이터 접근을 가능하게 만드는 관리 단위**이다.

---

## 1. 1차 캐시 (First-Level Cache)

### 역할

영속성 컨텍스트는 관리 중인 엔티티를 **내부 캐시(1차 캐시)** 에 저장한다.
- 같은 트랜잭션 내 **중복 DB 조회 방지**
- 조회 성능 최적화
- 트랜잭션 종료 시 함께 소멸

---

### 동작 흐름

1. 엔티티 조회 요청
2. 1차 캐시 탐색
3. 존재 → 캐시 반환
4. 없음 → DB 조회 후 캐시에 저장

---

## 2️. 동일성 보장 (Identity Guarantee)

### 역할

영속성 컨텍스트는 **같은 엔티티를 항상 같은 객체 인스턴스로 반환**한다.
- 값 동등성(`equals`)이 아닌, 객체 동일성(`==`) 보장
- 객체 그래프 탐색 안정성

---

### 예시

```java
Membera= em.find(Member.class,1L);
Memberb= em.find(Member.class,1L);

a == b// true

```

---

## 3️. 쓰기 지연 (Write-Behind / Buffered Write)

### 역할

INSERT / UPDATE / DELETE SQL을 즉시 실행하지 않고 **트랜잭션 커밋 시점까지 모아서 실행**한다.
- DB I/O 최소화
- 트랜잭션 단위 SQL 제어
- 성능 최적화
- 트랜잭션 단위 일관성 유지

---

### 동작 흐름

1. 엔티티 저장/삭제
2. SQL을 쓰기 지연 저장소에 보관
3. 커밋 시점에 일괄 실행

---

## 4️. 변경 감지 (Dirty Checking)

### 역할

엔티티의 상태 변화를 자동으로 감지하여 **트랜잭션 커밋 시 UPDATE SQL을 생성**한다.
- 객체 중심의 수정 로직
- 명시적인 UPDATE 호출 제거
- 영속 상태 엔티티만 대상
- 커밋 또는 flush 시점에 동작
---


### 동작 흐름

1. 엔티티 조회 → 초기 스냅샷 저장
2. 엔티티 값 변경
3. 커밋 시점에 스냅샷과 비교
4. 변경 감지 → UPDATE 실행

---

## 5️. 지연 로딩 (Lazy Loading)

### 역할

연관된 엔티티를 **실제 사용 시점까지 조회하지 않음**으로써 불필요한 DB 접근을 지연시킨다.
- 초기 조회 성능 최적화
- 필요할 때만 연관 엔티티 로딩
- 영속성 컨텍스트가 살아 있어야 동작
- 기본 필드는 지연 로딩 대상 아님
---

### 동작 흐름

1. 엔티티 조회 시 연관 엔티티는 프록시로 대체
2. 연관 엔티티 접근
3. 실제 DB 조회 실행

---

## 💻 코드 실습 및 분석
```java
// EntityManagerFactory 생성 (애플리케이션 시작 시 1번)
EntityManagerFactory emf =
        Persistence.createEntityManagerFactory("jpaemf");

// EntityManager 생성 (트랜잭션 단위)
EntityManager em = emf.createEntityManager();

// 트랜잭션 시작
EntityTransaction tx = em.getTransaction();
tx.begin();

try {
    // 비영속 상태 (단순 자바 객체)
    Member member = new Member("kim");

    // 영속 상태로 전환
    em.persist(member);

    tx.commit();
} catch (Exception e) {
    tx.rollback();
} finally {
    em.close(); // 영속성 컨텍스트 종료
}

emf.close();

```

---

## 💡 회고 및 질의응답
- 스프링이 결합된 형태보다는, 순수 JPA의 동작 과정에 대해 학습했다. JPA가 Repeatable Read를 보장하는 원리와 dirty check가 작동하는 방식에 대해 이해할 수 있었다. 특히, 개발 과정에서 Update 메서드를 작성할때 객체의 상태를 변경하고 save()를 다시 호출하지 않는 데에 대한 불안감이 있었는데, dirty check의 작동 원리를 알고 나니 오히려 불필요한 코드였다는 점을 알 수 있었다.
- 그동안 작성했던 코드를 다시 읽어보고, 리팩토링을 진행해보면 좋을 것 같다. 
