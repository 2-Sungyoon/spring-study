# 📅 [week 7] 주제: 인가 코드 탈취 방어(PKCE)와 실전 REST API 보안

## 🎯 학습 목표
- 인가 코드 가로채기(Interception Attack)의 원리와 이를 방어하는 PKCE 메커니즘 이해
- 세션(Session)과 토큰(Token) 기반 로그인 방식의 구조적 차이 및 REST API 필수 보안 요소 파악
- Adobe API 등 글로벌 기업의 명세에 명시된 PKCE 동작 파라미터 규격 학습

---

## 📝 주요 개념 정리

### 1. 인가 코드 가로채기(Interception Attack)란?
기존 OAuth 2.0 권한 부여 코드 승인 방식에서 모바일 앱이나 SPA(Single Page Application)가 가진 구조적 취약점임.
- 원리: 모바일 환경에서는 특정 앱을 실행하기 위해 커스텀 URI 스킴(Custom URI Scheme)을 사용함. 악성 앱이 정상적인 앱과 동일한 스킴을 등록해 두면, 인가 서버(카카오, 구글 등)가 로그인을 마치고 인가 코드를 리다이렉트할 때 악성 앱이 이 코드를 중간에서 가로챌 수 있음.
- 문제점: SPA나 모바일 앱 같은 공개 클라이언트(Public Client)는 백엔드 서버처럼 Client Secret을 안전하게 숨겨둘 수 없음. 악성 앱이 탈취한 인가 코드만으로도 인가 서버에 토큰을 요청하여 정상적으로 발급받아버리는 보안 사고가 발생함.

### 2. PKCE (Proof Key for Code Exchange) 방어 메커니즘
위와 같은 가로채기 공격을 막기 위해 도입된 OAuth 2.0 보안 확장 표준(RFC 7636)임.
- 동작 원리:
  1. 클라이언트가 로그인 요청 시 임의의 고유 난수(code_verifier)를 생성하고, 이를 해시 암호화한 값(code_challenge)을 카카오 인가 서버로 전송함.
  2. 인가 서버는 해당 해시값을 기억해 두고 인가 코드를 반환함.
  3. 클라이언트가 인가 코드를 토큰으로 교환할 때, 최초에 만든 원본 난수(code_verifier)를 함께 보냄.
  4. 인가 서버는 자신이 가진 해시값과 새로 받은 난수를 대조하여 일치할 때만 토큰을 발급함.
- 핵심 파라미터 규격 (S256 방식):
  - code_verifier: 클라이언트가 임의로 생성한 고유 난수 문자열.
  - code_challenge: verifier를 SHA-256 알고리즘으로 해싱한 뒤 Base64 URL Safe 인코딩한 값.
  - code_challenge_method: S256 (SHA-256 해싱을 사용함을 명시).
### 3. 세션(Session) vs 토큰(Token) 인증 방식 비교
- 세션 기반 인증 (Stateful): 서버 메모리나 Redis에 유저 상태를 저장함. 특정 유저 강제 로그아웃이나 동시 접속 제어(중복 로그인 방지)가 완벽하게 가능함. 단, 다중 서버 환경(Scale-out) 시 세션 클러스터링 동기화 등 아키텍처 관리가 까다로움.
- 토큰 기반 인증 (Stateless): JWT를 발급하여 클라이언트가 보관함. 서버 부하가 적고 확장이 자유로우나, 한 번 발급된 액세스 토큰은 만료 전까지 강제 무효화가 사실상 불가능함. 이를 보완하기 위해 리프레시 토큰 분리 및 HttpOnly 쿠키 적용이 필수적임.

### 4. 실전 REST API 보안 필수 고려사항
백엔드 개발자가 프레임워크에 의존하지 않고 직접 통제해야 하는 보안 요소임.
- CORS (Cross-Origin Resource Sharing) 엄격한 제어: API 통신 오류를 막기 위해 allowedOrigins("*") 설정으로 모든 출처를 허용하는 것은 보안상 치명적임. 반드시 연동된 프론트엔드 도메인만 명시적으로 허용하여 외부 악성 사이트의 API 호출을 방어해야 함.
- Rate Limiting (처리율 제한): 무차별 대입 공격이나 DDoS를 방어하기 위해 특정 IP나 유저별로 일정 시간 내 API 호출 횟수를 제한해야 함.
- SQL Injection 및 XSS 방어: JPA 파라미터 바인딩으로 SQL 인젝션은 1차 방어가 되나, 사용자 입력값이 그대로 응답 바디로 나갈 경우 XSS 취약점이 됨. 외부 입력값은 반드시 이스케이프(Escape) 처리를 거쳐야 함.
---


## 💻 코드 실습 및 분석

자바 백엔드 또는 프론트엔드에서 PKCE 검증을 위한 challenge를 생성하는 유틸리티 코드 구현임.

```java
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.util.Base64;
import java.util.UUID;

public class PkceUtil {
    
    // 1. code_verifier 생성 (최소 43자 이상 권장)
    public static String generateCodeVerifier() {
        // 보안 강도를 높이기 위해 UUID를 결합하여 난수 생성
        return UUID.randomUUID().toString() + UUID.randomUUID().toString();
    }

    // 2. verifier를 해시 암호화하여 code_challenge 생성 (S256 방식)
    public static String generateCodeChallenge(String verifier) throws Exception {
        byte[] bytes = verifier.getBytes(StandardCharsets.US_ASCII);
        
        // SHA-256 해시 알고리즘 적용
        MessageDigest md = MessageDigest.getInstance("SHA-256");
        byte[] digest = md.digest(bytes);
        
        // Base64 URL Safe 인코딩 및 패딩(=) 제거 (표준 규격)
        return Base64.getUrlEncoder().withoutPadding().encodeToString(digest);
    }
}
```
- 분석: 생성된 verifier는 클라이언트 내부 메모리에만 존재하며 외부 네트워크를 타지 않음. 외부 통신망으로는 단방향 암호화된 challenge만 전송되므로, 중간자 공격(MITM)으로 패킷을 탈취하더라도 역산하여 원본 verifier를 알아내는 것은 불가능함.

---

## 💡 회고 및 질의응답
- 6주차에 다룬 State 파라미터가 요청 위조(CSRF)를 막는 용도라면, PKCE는 인가 코드 가로채기를 암호학적으로 차단하는 기술임을 구별하여 이해함. Adobe 등 대형 IT 기업들이 API 연동 시 왜 해당 규격들을 필수로 강제하는지 설계적 근거를 명확히 파악함.

- Q. PKCE에서 code_challenge_method로 plain을 사용하면 안 되는 이유는 무엇인가?

  - plain 방식은 verifier를 해싱하지 않고 그대로 challenge로 사용하는 방식임. 이는 SHA-256을 지원하지 못하는 극단적인 레거시 환경을 위한 설정일 뿐임. plain을 사용하면 네트워크 단에 원본 난수가 그대로 노출되므로 PKCE를 적용하는 의미가 완전히 사라짐. 따라서 무조건 S256 방식을 사용해야 함.

- Q. 프론트와 백엔드가 분리된 현재 프로젝트 구조에서는 누가 verifier를 생성해야 하는가?

  - 카카오 인증 페이지로 최초 접근(리다이렉트)을 트리거하는 측에서 생성해야 함. 프론트엔드가 로그인 창을 띄운다면 프론트엔드가 생성하여 challenge를 카카오로 보내고, verifier는 세션 스토리지에 임시 보관함. 이후 카카오로부터 받은 인가 코드와 보관해둔 verifier를 백엔드로 함께 넘겨주어, 백엔드가 최종 토큰 발급 통신을 수행하도록 설계하는 것이 정석임.
