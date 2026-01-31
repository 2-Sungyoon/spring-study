# 📅 [week 2] 주제: JPA(Java Persistence API) 기본 정리

## 🎯 학습 목표
- JPA 주요 개념 및 정의 학습
- 연관관계 매핑 및 Fetch 전략 학습
- JPQL 개념 학습

---

## 📝 주요 개념 정리

## 1. JPA 개요

### 1.1 정의

- JPA(Java Persistence API)는 **자바 객체와 관계형 데이터베이스 간의 매핑을 정의한 표준 명세**
- ORM(Object-Relational Mapping) 기술의 표준 인터페이스
- JPA 자체는 구현체가 아니며, 대표적인 구현체는 **Hibernate**

### 1.2 목적

- 객체 중심의 도메인 모델을 유지하면서 DB 접근
- SQL 중심 개발의 한계를 극복
- 데이터 접근 로직의 추상화

---

## 2. 핵심 구성 요소

| 구성 요소 | 설명 |
| --- | --- |
| Entity | DB 테이블과 매핑되는 객체 |
| EntityManager | 영속성 컨텍스트를 관리하는 인터페이스 |
| Persistence Context | 엔티티를 관리하는 논리적 저장소 |
| Transaction | 데이터 변경의 단위 |
| JPQL | 엔티티 기반 객체 지향 쿼리 언어 |

---

## 3. 엔티티(Entity)

### 3.1 엔티티 정의

```java
@Entity
publicclassMember {
@Id
@GeneratedValue
private Long id;
}

```

### 3.2 엔티티 필수 조건

- 기본 생성자 필수 (접근 제한자: `protected` 권장)
- `@Entity` 선언
- 식별자 필드에 `@Id` 적용
- final 클래스, final 필드 사용 불가

### 3.3 권장 설계

- Setter 최소화
- 생성자 또는 의미 있는 변경 메서드 사용
- 비즈니스 로직 포함 가능

---

## 4. 영속성 컨텍스트 (Persistence Context)

### 4.1 정의

- 엔티티를 **영속 상태로 관리하는 논리적 공간**
- EntityManager를 통해 접근

### 4.2 엔티티 생명주기

| 상태 | 설명 |
| --- | --- |
| 비영속 | 영속성 컨텍스트와 무관 |
| 영속 | 영속성 컨텍스트에 의해 관리 |
| 준영속 | 영속 상태에서 분리 |
| 삭제 | 삭제 예약 상태 |

### 4.3 주요 기능

- 1차 캐시
- 동일성 보장 (`==`)
- 변경 감지(Dirty Checking)
- 쓰기 지연(SQL 지연 실행)

---

## 5. 트랜잭션(Transaction)

### 5.1 기본 개념

- JPA의 모든 데이터 변경 작업은 트랜잭션 내부에서 수행
- 트랜잭션 커밋 시점에 SQL 실행

### 5.2 Spring 환경

```java
@Transactional
publicvoidupdate() {
    entity.changeValue();
}

```

---

## 6. 연관관계 매핑

### 6.1 연관관계 방향

- 단방향
- 양방향 (객체 간 참조가 양쪽에 존재)

> 관계형 DB에는 방향 개념이 없음
> 
> 
> 방향은 객체 모델에서만 의미를 가짐
> 

### 6.2 연관관계의 주인

- 외래 키(FK)를 관리하는 쪽
- `mappedBy`가 없는 쪽이 주인

```java
@ManyToOne
@JoinColumn(name = "member_id")
private Member member;

```

### 6.3 설계 원칙

- 연관관계의 주인은 항상 `ManyToOne` 쪽
- 양방향 관계는 필요할 때만 사용

---

## 7. Fetch 전략

### 7.1 종류

| 전략 | 설명 |
| --- | --- |
| LAZY | 실제 사용 시 조회 |
| EAGER | 즉시 조회 |

```java
@ManyToOne(fetch = FetchType.LAZY)

```


---

## 8. Cascade & Orphan Removal

### 8.1 Cascade

```java
@OneToMany(cascade = CascadeType.ALL)

```

- 부모 엔티티의 생명주기를 자식에게 전파

### 8.2 Orphan Removal

```java
orphanRemoval =true

```

- 컬렉션에서 제거된 자식 엔티티 자동 삭제

### 8.3 사용 원칙

- Aggregate Root 내부에서만 사용
- 독립 생명주기를 갖는 엔티티에는 사용 금지

---

## 9. JPQL

### 9.1 특징

- 테이블이 아닌 **엔티티 기준 쿼리**
- SQL과 유사한 문법
- DB 독립성 제공

```sql
SELECT mFROMMember mWHERE m.name= :name

```

### 9.2 주의 사항

- JOIN 시 엔티티 관계 기반
- 컬럼명이 아닌 필드명 사용

---

## 10. Spring Data JPA

### 10.1 Repository

```java
publicinterfaceMemberRepositoryextendsJpaRepository<Member, Long> {
    Optional<Member>findByEmail(String email);
}

```

### 10.2 제공 기능

- CRUD 자동 구현
- 페이징 및 정렬
- 쿼리 메서드
- @Query 지원

---

## 💡 회고 및 질의응답
- 기존에는 Spring Data JPA를 활용해 CRUD와 연관관계 매핑을 구현하는 데에는 익숙했지만, 그 내부에서 실제로 어떤 일이 일어나는지에 대해서는 상대적으로 단편적으로 이해하고 있었다.
- JPA의 중심은 Repository나 EntityManager가 아니라 영속성 컨텍스트라는 점이 인상깊었다. 특히 엔티티의 필드를 변경했을 때 UPDATE 쿼리가 실행되는 이유가 트랜잭션 커밋 시점의 변경 감지 메커니즘 때문이라는 점을 알게 되는 등, 그동안 막연하게 사용하던 JPA의 동작 흐름을 어느정도 이해할 수 있게 되었다.
