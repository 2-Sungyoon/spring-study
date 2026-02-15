# 📅 [week 4] 주제: JPA 객체–테이블 매핑

## 🎯 학습 목표
- 객체와 관계형 데이터베이스 테이블 간의 매핑
- 객체를 통해 연관관계 표현

---

## 📝 주요 개념 정리
# 객체–테이블 매핑 정리 (JPA)

## 1. 기본 매핑 개요

### 1.1 목적

- 객체 모델과 관계형 데이터베이스 테이블 간의 불일치 해결
- 객체 지향적 설계를 유지하면서 DB 무결성 확보

### 1.2 핵심 원칙

- **엔티티 = 테이블**
- **엔티티 필드 = 컬럼**
- **연관관계 = 외래키(FK)**

---

## 2. 엔티티 기본 매핑

### 2.1 엔티티 선언

```java
@Entity
@Table(name = "member")
publicclassMember {
}

```

### 2.2 기본 키 매핑

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

```

| 전략 | 특징 |
| --- | --- |
| IDENTITY | DB Auto Increment |
| SEQUENCE | DB 시퀀스 사용 |
| TABLE | 키 테이블 사용 |
| AUTO | JPA가 DB에 맞게 선택 |

---

## 3. 컬럼 매핑

```java
@Column(name = "email", nullable = false, length = 100, unique = true)
private String email;

```

### 주요 속성

- `name` : 컬럼명
- `nullable` : NOT NULL 여부
- `unique` : 유니크 제약
- `length` : 문자열 길이
- `columnDefinition` : 직접 DDL 지정

### 주의점

- 제약조건은 테이블 생성시에만 동작
- ddl관련 옵션 설정에 의해 동작하고, 해당 옵션 사용시 테이블이 삭제될 위험이 있으므로 제약조건 관련 옵션을 사용하기보다 데이터베이스에서 직접 테이블 생성하는 것을 권장

---

## 4. 연관관계 매핑 개요

### 4.1 관계 유형

- 1:1
- 1:N
- N:1
- N:M (실무에서는 중간 테이블로 분해)

### 4.2 핵심 개념

- **연관관계의 주인**
    - 외래키(FK)를 실제로 관리하는 엔티티
    - DB 컬럼을 기준으로 결정
- **mappedBy**
    - 연관관계의 주인이 아님
    - 읽기 전용(객체 그래프 탐색용)

---

## 5. N:1 / 1:N 매핑

### 5.1 권장 기본 구조 (단방향)

```java
@Entity
publicclassPost {

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "member_id")
private Member member;
}

```

- 외래키는 항상 **N쪽 테이블**
- @OneToMany로 주인쪽에서 매핑하면 의도하지 않은 update 쿼리가 발생하므로, 실무에서 사용을 권장하지 않음(mappedBy옵션으로 양방향 매핑하는경우가 아닌, 단방향에서 OneToMany 사용하는 경우를 의미)

---

### 5.2 양방향 매핑

```java
@Entity
publicclassMember {

@OneToMany(mappedBy = "member")
private List<Post> posts =newArrayList<>();
}

```

```java
@Entity
publicclassPost {

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "member_id")
private Member member;
}

```

### 규칙

- 연관관계 주인: `Post.member`
- `Member.posts`는 DB 컬럼 생성 하지 않음
- 컬렉션은 조회/탐색 용도
- 처음부터 양방향 매핑을 하기보다, 필요시에만 생성
    - 컬렉션 관리가 복잡해지고, N+1 문제 발생 가능성을 최소화하기 위함


## 💡 회고 및 질의응답
- 데이터베이스를 자바 코드를 통해 엔티티로 변환할 때 M:N 관계를 표현하는 방법에 대해 고민이 많았다. 테이블과 객체를 매핑하는 원리를 학습하고 나니, 별도 테이블을 분리함으로써 얻을 수 있는 추가 필드의 이점 등의 이유로 거의 대부분의 경우 @ManyToMany를 사용해서 얻는 이득이 없다고 느껴졌다.
- @OneToMany와 @ManyToOne의 내부 동작 차이를 통해 JPA에 의해 발생하는 의도치 않은 쿼리를 최소화하는 방법을 알 수 있었고, 특히 컬렉션을 어떤 경우에 사용하는지 의문이 있었는데, 필요한 경우가 아니면 컬렉션 사용이 오히려 문제를 가져올 확률이 높아지는 것 같다.
