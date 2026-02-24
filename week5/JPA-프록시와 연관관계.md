# 📅 [week 5] 주제: JPA-프록시와 연관관계

## 🎯 학습 목표
- JPA에서 프록시 객체가 활용되는 원리를 학습한다.
- 연관관계에서 FetchType, Cascade, orphanRemoval 등의 개념 대해 학습한다.

---

## 📝 주요 개념 정리
## 1. 프록시(Proxy)

### ■ 개념

프록시는 실제 엔티티 객체를 즉시 조회하지 않고, **대리 객체를 먼저 반환**한 뒤 실제 데이터 접근 시점에 조회하도록 하는 기술이다.

JPA는 지연 로딩을 구현하기 위해 프록시 객체를 사용한다.

---

### ■ 동작 흐름

```
Membermember=em.find(Member.class,1L);
Teamteam=member.getTeam();// 프록시 반환
team.getName();// 실제 조회 실행
```

1. 연관 객체 대신 프록시 반환
2. 프록시 내부에 식별자 보관
3. 실제 데이터 접근 시 SELECT 실행

---

### ■ 특징

- 실제 엔티티를 상속한 대리 객체
- 초기에는 식별자만 보유
- 최초 접근 시 DB 조회 수행

---

### ■ 확인 방법

```
team.getClass()
```

출력 예:

```
class Team$HibernateProxy
```

---

### ■ 주의사항

- 영속성 컨텍스트 종료 후 접근 시 LazyInitializationException 발생

- equals(), instanceof 사용 시 주의 필요

- 초기화 여부 확인 가능

```
Hibernate.isInitialized(team);
```

---

## 2. 지연 로딩 (LAZY Loading)

### ■ 개념

연관 엔티티를 **실제 사용하는 시점에 조회**하는 전략이다.

---

### ■ 동작 방식

```
@ManyToOne(fetch=FetchType.LAZY)
private Team team;
```

### 실행 흐름

1. Member 조회
2. Team은 조회하지 않음
3. team.getName() 호출 시 SELECT 실행

---

### ■ SQL 흐름

```
SELECT*FROM member;

-- 실제 접근 시
SELECT*FROM teamWHERE id= ?;
```

---

### ■ 장점

- 불필요한 데이터 로딩 방지

- 성능 예측 가능

- 메모리 사용 절감

---

### ■ 단점

- N+1 문제 발생 가능

- 영속성 컨텍스트 종료 후 접근 시 예외 발생

---

### ■ 사용 권장 상황

- 목록 조회
- 대량 데이터 조회
- 연관 데이터 사용 여부가 불확실한 경우
- 대부분의 경우 LAZY를 권장

---

## 3. 즉시 로딩 (EAGER Loading)

### ■ 개념

엔티티 조회 시 연관 엔티티를 **즉시 함께 조회**하는 전략이다.

---

### ■ 설정

```
@ManyToOne(fetch=FetchType.EAGER)
```

---

### ■ SQL 예시

```
SELECT m.*, t.*
FROM member m
JOIN team tON m.team_id= t.id;
```

---

### ■ 장점

- 추가 조회 발생 없음

- 코드 단순화

---

### ■ 단점

- 불필요한 JOIN 발생

- 성능 예측 어려움

- 메모리 사용 증가

- 연관 관계 확장 시 JOIN 폭발 가능

- JPQL 사용 시 N+1 문제 발생 가능

---

### ■ 사용 권장 상황

- 항상 함께 사용되는 작은 연관 객체
- 대부분의 경우 EAGER 사용을 지양

---

## 4. Cascade (영속성 전파)

### ■ 개념

부모 엔티티의 영속성 상태 변화가 자식 엔티티로 전파되는 기능이다.

---

### ■ 주요 옵션

| 옵션 | 동작 |
| --- | --- |
| PERSIST | 저장 전파 |
| REMOVE | 삭제 전파 |
| MERGE | 병합 전파 |
| REFRESH | 새로고침 전파 |
| DETACH | 분리 전파 |
| ALL | 모든 전파 |

---

### ■ 예시

```
@OneToMany(cascade=CascadeType.PERSIST)
privateList<OrderItem>items;
```

```
orderRepository.save(order);
```

→ OrderItem 자동 저장

---

### ■ 동작 흐름

1. 부모 persist 호출
2. cascade 설정 확인
3. 자식 persist 전파
4. flush 시 INSERT 실행

---

### ■ 사용 기준

- 부모가 자식의 생명주기를 완전히 관리할 때

- 자식이 독립적으로 존재할 의미가 없을 때

예:

- 주문 → 주문항목
- 장바구니 → 항목
- 게시글 → 첨부파일

---

### ■ 사용 주의

DB 레벨에서의 Cascade 옵션이 아니라, 연관관계의 생명주기를 관리하는 옵션이다.

---

## 5. orphanRemoval

### ■ 개념

부모 엔티티의 컬렉션에서 제거된 자식 엔티티를 **자동 삭제**하는 기능이다.

---

### ■ 설정

```
@OneToMany(orphanRemoval=true)
```

---

### ■ 동작 예시

```
order.getItems().remove(item);
```

→ DB에서 item DELETE 실행

---

### ■ Cascade REMOVE와 차이

| 기능 | 동작 |
| --- | --- |
| REMOVE | 부모 삭제 시 자식 삭제 |
| orphanRemoval | 부모 컬렉션에서 제거 시 삭제 |

---

### ■ 사용 조건

- 부모가 자식을 소유하는 관계

- 자식 단독 존재 의미 없음

---

### ■ 주의사항

- 단순 관계 해제 목적이라면 사용 금지
- 실수로 컬렉션 제거 시 데이터 삭제 발생 가능
---

## 💡 회고 및 질의응답
- 데이터베이스에서 cascade 옵션 설정이 필요한 상황에서 @ManyToOne 등의 연관관계 매핑을 위한 어노테이션에 cascade옵션 사용이 가능하다고 생각했는데, 해당 옵션이 DB 레벨에서의 cascade 적용이 아니라 영속성 전파에 관련된 옵션이라는 사실을 알게 되었다. 실제로 해당 옵션의 enum 타입을 확인해보면, ondelete와 같은 옵션이 존재하지 않음을 확인할 수 있다.
- fetch 전략에 대해 LAZY를 권장하는 이유에 대해 단순히 조인 비용을 절약하고 불필요한 로딩을 피할 수 있기 때문이라고 생각했으나, JPQL 사용 시 별도의 SQL을 생성하는 것을 확인할 수 있었다.
