# 📅 [week 7] 주제: 인가 코드 탈취 방어(PKCE) & Polyglot Persistence

## 🎯 학습 목표
- 인가 코드 가로채기(Interception Attack)의 원리와 이를 방어하는 글로벌 보안 표준인 PKCE 메커니즘을 이해
- 단일 데이터베이스의 한계를 극복하는 '폴리글랏 퍼시스턴스(Polyglot Persistence)' 개념을 학습하고, 인증 시스템에서 이중 DB(RDBMS + NoSQL)를 도입하는 논리적 타당성을 검증.
- 토큰 기반 인증의 구조적 한계를 보완하는 실무적 방어 기법(토큰 분리 전송, RTR) 및 REST API 필수 보안 요소를 파악.

---

## 📝 주요 개념 정리

### 1. 6주차 핵심 요약: 웹 표준 보안의 적용
- **XSS 및 CSRF 방어 구조:** 수명이 긴 Refresh Token은 자바스크립트 접근이 불가능한 `HttpOnly` 및 `Secure` 속성의 쿠키로 격리하여 XSS를 방어합니다. 수명이 짧은 Access Token은 JSON 응답 바디로 전달하여 클라이언트 메모리에서 관리하게 함으로써 CSRF 공격 피해를 최소화하는 토큰 분리 전략을 적용했습니다.

### 2. 인가 코드 가로채기(Interception Attack)와 PKCE 방어 메커니즘

기존 OAuth 2.0 권한 부여 코드 승인 방식에서 모바일 앱이나 SPA(Single Page Application) 같은 공개 클라이언트(Public Client)가 가진 구조적 취약점과 그 해결책입니다.

- **가로채기 공격 원리:** 모바일 환경에서는 특정 앱을 실행하기 위해 커스텀 URI 스킴(Custom URI Scheme)을 사용합니다. 악성 앱이 정상적인 앱과 동일한 스킴을 OS에 등록해 두면, 인가 서버(카카오, 구글 등)가 로그인을 마치고 인가 코드를 리다이렉트할 때 악성 앱이 이 코드를 중간에서 가로챌 수 있습니다. 백엔드 서버와 달리 Client Secret을 안전하게 숨길 수 없는 환경이므로, 탈취된 인가 코드만으로도 토큰이 정상 발급되는 대형 보안 사고가 발생합니다.
- **PKCE (Proof Key for Code Exchange) 도입:** 이 취약점을 암호학적으로 차단하기 위해 도입된 OAuth 2.0 보안 확장 표준(RFC 7636)입니다.
  1. 클라이언트가 로그인 요청 시 임의의 고유 난수(`code_verifier`)를 생성하고, 이를 단방향 해시 암호화한 값(`code_challenge`)만 카카오 인가 서버로 전송합니다.
  2. 인가 서버는 해당 해시값을 기억해 두고 인가 코드를 반환합니다.
  3. 클라이언트가 인가 코드를 토큰으로 교환할 때, 최초에 만든 원본 난수(`code_verifier`)를 함께 보냅니다.
  4. 인가 서버는 자신이 가진 해시값과 새로 받은 난수를 대조하여 일치할 때만 토큰을 발급합니다.
- 네트워크에는 단방향 암호화된 `challenge`만 전송되므로, 중간자 공격(MITM)으로 패킷을 탈취하더라도 역산하여 원본 `verifier`를 알아내는 것은 계산적으로 불가능(Computationally Infeasible)합니다.

### 3. 인증 아키텍처의 패러다임 전환: 폴리글랏 퍼시스턴스 (Polyglot Persistence)

단순히 로그인을 구현하는데 왜 메인 RDBMS(PostgreSQL) 외에 보조 DB(Redis, MongoDB)를 두어야 하는지에 대한 아키텍처적 근거입니다.

- **시대적 배경과 개념:** 모든 데이터를 하나의 관계형 데이터베이스(RDBMS)로 처리하지 않고, 데이터의 성격과 처리 방식에 가장 적합한 도구를 선택해야 한다는 인식이 확산되었고, 여러 종류의 데이터 저장 기술을 조합하여 사용하는 전략인 **'폴리글랏 퍼시스턴스'**가 대두되었습니다. 
- **마이크로서비스(MSA)와 데이터베이스 분리:** 거대한 플랫폼을 예로 들면, 결제나 주문 서비스는 데이터의 정합성(ACID)이 생명인 RDBMS를 사용해야 합니다. 반면, 상품 카탈로그는 스키마가 유연한 Document DB를, 유저 간 관계 탐색은 Graph DB를 사용하는 것이 효율적입니다.
- **로그인/세션 관리에 보조 DB를 사용하는 이유:** 유저의 정보와 비밀번호는 RDBMS에 저장하는 것이 맞습니다. 하지만 **토큰(특히 Refresh Token)은 성격이 다릅니다.**
  1. **극단적인 I/O 발생:** 토큰 갱신 시마다 발생하는 엄청난 읽기/쓰기 부하를 RDBMS에 주면, 정작 중요한 비즈니스 로직(결제 등)에 병목이 발생합니다.
  2. **데이터의 휘발성 (Dead Data):** 만료된 토큰은 쓰레기 데이터입니다. RDBMS는 이를 자동으로 지워주지 않아 배치 스케줄러가 필요하지만, 인메모리 기반의 **Redis**나 Document 기반의 **MongoDB**는 DB 엔진 단에서 제공하는 **TTL(Time-To-Live) 기능**을 통해 수명이 다한 데이터를 메모리에서 알아서 삭제힙니다.

### 4. Redis vs MongoDB와 다중 기기 제어
폴리글랏 퍼시스턴스 전략에 따라 로그인 토큰 저장소를 선택할 때, 실무에서는 비즈니스 요구사항에 맞춰 최적의 도구를 선택합니다.

- **Redis:** 응답 속도가 관건인 세션 관리에 가장 보편적으로 사용되는 Key-Value 저장소입니다. 압도적으로 빠르지만 구조가 단순합니다.
- **MongoDB:** 현대 서비스는 PC, 모바일 등 다중 기기 동시 로그인을 지원해야 합니다. 즉, 유저 1명당 N개의 기기별 토큰이 필요합니다. MongoDB는 유연한 문서(Document) 구조를 가지므로, 토큰 값뿐만 아니라 **접속 IP, User-Agent(브라우저/기기 정보), 생성 시간 등을 하나의 객체로 묶어 저장**하는 데 최적화되어 있습니다. 이를 통해 '다른 모든 기기에서 로그아웃'이나 '비정상 접속 탐지'와 같은 고도화된 보안 기능을 논리적으로 완벽하게 구현할 수 있습니다.
<img width="714" height="223" alt="image" src="https://github.com/user-attachments/assets/05d6bbed-3c46-4bb6-bc79-76c0d519634a" />
- 성능을 위해 Redis를 1차 캐시 및 세션 저장소로 활용하고, MongoDB를 유저의 로그인 이력 및 기기 관리용 영속성 저장소로 두는 계층형 폴리글랏 구조를 취하기도 합니다.

### 5. 실전 REST API 보안 필수 고려사항
- **토큰 분리 저장 (XSS / CSRF):** 프론트엔드의 구조적 취약점을 막기 위해, 수명이 긴 리프레시 토큰은 자바스크립트 접근이 OS 단에서 차단되는 `HttpOnly 쿠키`로 전송합니다. 반면, 수명이 짧은 액세스 토큰은 응답 바디(JSON)로 전달해 메모리에만 저장함으로써 쿠키의 약점인 CSRF 공격을 무력화합니다.
- **RTR (Refresh Token Rotation):** 쿠키가 물리적으로 탈취되는 극단적인 상황을 방어하기 위해, Access Token 갱신 시마다 기존 Refresh Token을 즉시 폐기하고 새로운 토큰 세트를 발급하는 실무적 방어 패턴입니다.
  - 즉 이미 사용된(Old) 리프레시 토큰이 다시 들어올 경우 이를 '탈취 사고'로 간주하는 것 [해당 유저와 연관된 모든 세션을 즉시 무효화(Revoke)]
- **기본 보안 통제:** 무차별 대입 방지를 위한 Rate Limiting(처리율 제한), 허용된 프론트엔드 도메인만 통신하게 하는 엄격한 CORS 제어, 사용자 입력값을 이스케이프 처리하는 XSS/SQL Injection 방어는 프레임워크에 의존하지 않고 직접 통제해야 할 필수 요소입니다.

---

## 💻 코드 실습 및 분석

자바를 통해 PKCE 검증을 위한 challenge를 생성하는 유틸리티 코드 구현

```java
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.security.SecureRandom;
import java.util.Base64;

public class PkceUtil {
    
    // 1. code_verifier 생성 (RFC 7636: 암호학적으로 안전한 난수 사용 권장)
    public static String generateCodeVerifier() {
        // SecureRandom을 사용하여 보안 강도를 극대화
        // 보안 토큰 생성 시에는 예측 가능성이 있는 UUID보다 OS 레벨의 엔트로피를 사용하는 SecureRandom을 사용하는 것이 글로벌 보안 표준(Best Practice)에 부합
        SecureRandom secureRandom = new SecureRandom();
        byte[] codeVerifier = new byte[32]; // 256 bits: 긴 길이의 난수 생성
        secureRandom.nextBytes(codeVerifier);
        
        // Base64 URL Safe 인코딩 (표준 규격 준수)
        return Base64.getUrlEncoder().withoutPadding().encodeToString(codeVerifier);
    }

    // 2. verifier를 해시 암호화하여 code_challenge 생성: 즉 가로채더라도, 원본을 알 수 없게 변형
    public static String generateCodeChallenge(String verifier) {
        try {
            // SHA-256 알고리즘은 Java 표준 사양
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            byte[] digest = md.digest(verifier.getBytes(StandardCharsets.US_ASCII));
            
            return Base64.getUrlEncoder().withoutPadding().encodeToString(digest);
        } catch (NoSuchAlgorithmException e) {
            // 시스템에서 지원하지 않는 알고리즘일 경우 예외 처리 로직이 필요합니다.
            throw new RuntimeException("SHA-256 algorithm not found", e);
        }
    }
}
```
- 보안 토큰 생성 시에는 예측 가능성이 있는 UUID보다 OS 레벨의 엔트로피를 사용하는 SecureRandom을 사용하는 것이 글로벌 보안 표준(Best Practice)에 부합
- 분석: 생성된 verifier는 클라이언트의 내부 메모리에만 존재하며 외부 네트워크를 타지 않습니다. 외부 통신망으로는 단방향 암호화된 challenge만 전송되므로, 악성 앱이 패킷을 탈취하더라도 이 값으로부터 원본 verifier를 역산해 내는 것은 불가능합니다. 이를 통해 PKCE가 인가 코드 가로채기를 원천 차단합니다.
---

## 💡 회고 및 질의응답
- 기존에는 단순히 기능 구현에만 매몰되어 단일 DB와 로컬 스토리지를 무비판적으로 사용했습니다. 하지만 폴리글랏 퍼시스턴스의 개념을 도입하여 데이터를 목적에 맞게 분리(RDBMS + MongoDB)하고, 글로벌 표준 규격(PKCE, HttpOnly)을 적용함으로써 시스템의 성능, 확장성, 보안성이 향상됨을 논리적으로 체감했습니다. 

Q. PKCE에서 code_challenge_method로 plain을 사용하면 안 되는 이유는 무엇입니까?

plain 방식은 생성된 난수(verifier)를 해싱하지 않고 원본 그대로 challenge 파라미터로 전송하는 방식입니다. 이는 SHA-256을 도저히 지원할 수 없는 극단적인 레거시 기기를 위해 남겨둔 설정일 뿐입니다. 이를 사용하면 통신망에 원본 난수가 그대로 노출되므로 탈취 방어라는 PKCE의 존재 이유가 완전히 상실됩니다. 따라서 무조건 암호학적으로 안전한 S256 방식을 강제해야 합니다.

Q. 프론트와 백엔드가 분리된 현재 프로젝트 구조에서는 누가 PKCE verifier를 생성해야 합니까?

카카오 인증 페이지로 리다이렉트를 최초로 트리거하는 측(일반적으로 프론트엔드)에서 생성해야 합니다. 프론트엔드가 난수를 생성하여 카카오로 challenge를 넘기고, 원본 verifier는 세션 스토리지에 잠깐 보관합니다. 이후 콜백으로 인가 코드가 돌아오면, 보관해둔 verifier를 인가 코드와 함께 백엔드로 넘겨주어 백엔드가 최종적으로 안전하게 토큰 교환을 수행하도록 설계하는 것이 보안 표준에 부합합니다.

Q. 단순히 로그인을 구현하는데 MongoDB까지 띄우는 것은 오버 엔지니어링이 아닙니까?

단일 기기 접속만 허용하는 토이 프로젝트라면 RDBMS 하나로도 충분합니다. 하지만 실제 글로벌 서비스나 상용 환경에서는 모바일, PC 등 다중 기기 접속 제어와 유저 환경(User-Agent) 기록이 필수적으로 요구됩니다. 메인 DB 트랜잭션의 병목을 막고 토큰의 자동 만료 처리를 DB 엔진 단(TTL)에 위임하여 자원 효율성을 극대화하는 것은 오버 엔지니어링이 아니라, 전체 시스템의 장애를 예방하기 위한 보편적이고 타당한 아키텍처 설계(폴리글랏 퍼시스턴스)입니다.
