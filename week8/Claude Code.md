# 📅 [week 8] 주제: Claude Code 활용 — IntelliJ 연동 & 워크플로우

## 🎯 학습 목표

- Claude Code 핵심 기능 전반을 파악한다
- CLAUDE.md 기반 메모리 시스템으로 세션 간 컨텍스트를 유지하는 방법을 익힌다
- 커스텀 슬래시 커맨드와 서브에이전트를 직접 정의해 반복 작업을 자동화한다
- `/compact` 이후에도 맥락이 끊기는 근본 원인을 이해하고 대응 전략을 세운다
- `@파일 참조`, `!셸 명령어`, `#메모리` 단축 문법을 체득한다
- Java/Spring 프로젝트에 맞는 워크플로우 패턴을 수립한다

---

## 📝 주요 개념 정리

### 1. Claude Code란

Claude Code는 Anthropic이 만든 CLI 기반 에이전틱 코딩 어시스턴트다. 브라우저 Claude와 근본적으로 다른 점은, 파일을 읽고 수정하고 셸 명령을 실행하는 등 **에이전트로서 작동**한다는 것이다. 질의응답 도구가 아니라 개발 작업의 실행 주체다.

**IntelliJ 환경에서의 포지셔닝**

| 도구 | 역할 | 컨텍스트 범위 |
|---|---|---|
| IntelliJ 자동완성 | 한 줄 수준 제안 | 현재 파일 |
| Claude Code (플러그인) | 멀티 파일 에이전틱 작업 | 프로젝트 전체 |
| 브라우저 Claude | 질의응답 | 입력한 텍스트만 |

**vs 대체재 비교**

| | Claude Code | GitHub Copilot | Cursor |
|---|---|---|---|
| 컨텍스트 범위 | 프로젝트 전체 | 현재 파일 중심 | 프로젝트 전체 |
| 에이전틱 실행 | ✅ 파일 수정, 셸 실행 | ❌ | 제한적 |
| 커스터마이징 | CLAUDE.md, 커스텀 커맨드, Hooks | 제한적 | Rules 파일 |
| IDE 종속 | CLI 기반 (IDE 무관) | IDE 플러그인 | 자체 IDE 교체 필요 |
| 트레이드오프 | 터미널 친숙도 필요 | 진입장벽 낮음 | 설치형 IDE 변경 필요 |

---

### 2. 커맨드 체계

Claude Code 커맨드는 두 범주로 나뉜다.

- **CLI 플래그**: 세션 시작 시 터미널에서 전달 (`claude --model sonnet`). 세션 중 변경 불가 (단, `/model`과 `/permissions`는 예외)
- **슬래시 커맨드**: 세션 내부에서 입력 (`/help`). 실행 중 동작을 제어

**핵심 슬래시 커맨드**

```
/help        현재 세션에서 사용 가능한 모든 커맨드 확인 (커스텀 포함)
/init        프로젝트 스캔 후 CLAUDE.md 자동 생성
/model       모델 전환 (복잡한 설계 → Opus, 단순 작업 → Haiku)
/review      현재 변경사항 코드 리뷰 요청
/config      모델, 툴 권한, 설정 관리
/compact     컨텍스트 압축 (토큰 부족 시)
/clear       대화 초기화 (완전히 다른 태스크 전환 시)
/context     현재 세션 토큰 사용량 확인 (MCP 서버별 포함)
/mcp         연결된 MCP 서버 목록 및 관리
/agents      서브에이전트 관리 (생성/선택/삭제)
/rewind      특정 체크포인트로 대화/코드 상태 되돌리기
/exit        세션 종료
```

**실전 단축 문법 3가지**

**① `@` — 파일/디렉터리 참조**

```bash
# 특정 파일을 컨텍스트로 포함
@src/main/java/com/example/service/OrderService.java 이 클래스의 의존성을 분석해줘

# 디렉터리 전체 참조
@src/main/java/com/example/repository/ JPA 쿼리 메서드 네이밍 일관성 검토해줘
```

파일명을 정확히 모를 경우 Claude Code가 내부적으로 grep해서 찾는다.

**② `!` — 셸 명령 직접 실행**

```bash
!./gradlew test          # Claude 대화형 모드를 우회 → 토큰 소비 없이 실행
!./gradlew build
!git diff HEAD
```

**③ `#` — 즉시 메모리 추가**

```bash
# 트랜잭션은 Service 레이어에서만 선언, @Transactional(readOnly=true) 기본값
# 이 프로젝트는 멀티모듈 구조이며 core / api / infra 모듈로 분리됨
```

입력하는 즉시 CLAUDE.md에 저장된다.

---

### 3. CLAUDE.md — 프로젝트 메모리

CLAUDE.md는 프로젝트의 메모리 역할을 하는 마크다운 파일이다. 계층 구조로 동작해 홈 디렉터리, 프로젝트 루트, 하위 모듈별로 각각 설정할 수 있다.

```
~/.claude/CLAUDE.md          ← 모든 프로젝트에 적용 (개인 전역 설정)
  └── ./CLAUDE.md            ← 현재 프로젝트에 적용 (팀 공유)
        └── ./module/CLAUDE.md  ← 하위 모듈에 적용 (선택적)
```

**Spring 백엔드 프로젝트 예시**

```markdown
# 프로젝트 개요
- Java 17, Spring Boot 3.x, JPA/Hibernate, MySQL 8
- 멀티모듈: core(도메인), api(컨트롤러), infra(영속성)

# 아키텍처 규칙
- 의존 방향: api → core ← infra (core는 외부 의존 금지)
- 트랜잭션 경계: Service 레이어에서만 @Transactional 선언
- 엔티티는 불변 필드 최대화, setter 사용 금지
- 도메인 로직은 엔티티 내부에 캡슐화

# 코드 컨벤션
- 인터페이스 네이밍: 접미사 없이 (OrderRepository O, IOrderRepository X)
- 예외는 커스텀 예외 계층 사용 (BusinessException 상속)
- 테스트: @SpringBootTest 지양, 단위 테스트 우선

# 자주 쓰는 명령어
- 빌드: ./gradlew build
- 테스트: ./gradlew test
- 특정 모듈 테스트: ./gradlew :core:test

# [설계 결정 로그]
- 2026-04-07: 멀티모듈 구조 확정 (core/api/infra)
```

**트레이드오프**

| 구분 | 장점 | 단점 |
|---|---|---|
| CLAUDE.md 상세 작성 | 매 세션 컨텍스트 자동 주입, 팀 표준 공유 | 파일 커질수록 토큰 소비 증가 |
| CLAUDE.md 없이 사용 | 토큰 효율 | 매번 같은 설명 반복 |

> 핵심 규칙만 간결하게 유지하고 (20줄 이내 권장), 상세 문서는 별도 파일로 분리해 `@docs/architecture-rules.md` 형태로 필요 시 참조하는 방식이 낫다.

---

### 4. 컨텍스트 유지 문제 — 근본 원인과 대응

> 💬 *"GPT보다 맥락 유지가 안 되는 것 같다. `/compact` 이후든 같은 프로젝트 내에서든"*

이건 Claude Code의 설계적 특성에서 비롯된다. 원인을 알면 대응 방법이 보인다.

**원인 1: `/compact`는 요약이지 기억이 아니다**

`/compact`는 토큰 확보를 위해 대화를 압축 요약한다. 이 과정에서 세부 결정 사항, 논의 맥락, 코드 의도가 손실된다. Claude는 압축된 요약본만 가지고 이후 대화를 이어가므로 맥락이 흐려지는 것이다.

```bash
# ❌ /compact만 믿고 이어가기
/compact
"계속해줘"
# → Claude는 무엇을 기준으로 계속해야 하는지 모름

# ✅ /compact 직후 핵심 결정 사항을 수동으로 재공급
/compact
"지금까지 결정된 사항:
1. Order 엔티티에 정적 팩토리 메서드 도입
2. Service는 팩토리 호출만 담당, 비즈니스 로직은 엔티티에
3. 낙관적 락(@Version) 적용 완료
이 방향으로 PaymentService 구현 계속해줘"
```

**원인 2: 세션 간 컨텍스트는 자동으로 이어지지 않는다**

Claude Code는 세션이 끊기면 이전 대화를 기억하지 못한다. `~/.claude/projects/<project>/memory/`에 자동 저장되지만, 이것만으로는 충분하지 않다.

대응 전략은 **CLAUDE.md에 의사결정 로그를 남기는 것**이다. CLAUDE.md는 매 세션마다 자동으로 주입되므로 세션이 끊겨도 결정 맥락이 유지된다.

```markdown
# CLAUDE.md에 결정 로그 섹션 추가
## [설계 결정 로그]
- 2026-04-07: Saga 패턴 채택 (분산 트랜잭션 대신, 서비스 독립성 우선)
- 2026-04-07: Order 엔티티 정적 팩토리 메서드 도입 (Order.create(...))
- 2026-04-07: 낙관적 락(@Version) 전체 엔티티에 적용
```

**원인 3: 컨텍스트 오염 (Context Rot)**

대화가 길어질수록 초반에 설정한 제약조건이나 방향이 희석된다. 새 태스크로 전환할 때 `/clear`를 쓰지 않으면 이전 맥락이 방해가 된다.

```bash
# 판단 기준
같은 태스크 맥락 유지 + 토큰 부족   → /compact + 결정 사항 재주입
완전히 다른 태스크로 전환            → /clear
세션 재시작                          → CLAUDE.md의 결정 로그가 자동 주입됨
```

**Claude Code vs GPT의 근본적 차이**

Claude Code는 대화 기억보다 **파일(CLAUDE.md)을 통한 상태 관리**가 기본 철학이다. GPT가 대화 흐름에 의존하는 방식이라면, Claude Code는 파일 시스템 에이전트로 설계된 만큼 파일로 상태를 명시해줄수록 일관성이 높아진다.

---

### 5. 토큰 절약 전략

**토큰 소비 주범별 대응**

**① 불필요한 파일 전체 참조**

```bash
# ❌ 패키지 전체를 컨텍스트로 던짐
@src/main/java/com/example/ 전체 구조 분석해줘

# ✅ 목적 파일만 정확히 지정
@src/main/java/com/example/service/OrderService.java
@src/main/java/com/example/domain/Order.java
Order 생성 로직의 트랜잭션 경계만 검토해줘
```

**② 대화형 모드로 셸 실행**

```bash
# ❌ Claude가 해석 + 실행 + 응답 생성 (토큰 3배 소비)
"테스트 실행해줘"

# ✅ ! 로 직접 실행 (대화형 모드 우회)
!./gradlew test
```

**③ CLAUDE.md 비대화**

핵심 규칙만 CLAUDE.md에, 상세 문서는 별도 파일로 분리한다.

```markdown
# CLAUDE.md 안에는 참조만 명시
상세 규칙: @docs/architecture-rules.md
API 스펙: @docs/api-spec.md
```

**④ 토큰 현황 파악**

```bash
/context   # 현재 토큰 사용량 및 MCP 서버별 소비량 확인
```

**⑤ 모델 전략적 전환**

```bash
/model haiku   # 보일러플레이트, 단순 리팩터링
/model sonnet  # 일반 기능 구현, 버그 수정 (기본값)
/model opus    # 아키텍처 설계, 복잡한 트랜잭션 로직 분석
```

복잡한 설계 논의는 Opus로 하고, 구현 단계에서 Sonnet으로 내리는 것이 비용 대비 효과가 좋다.

---

### 6. 복잡한 Spring 코드 구조 이해시키기

Claude Code는 파일을 직접 읽으므로 "설명"보다 **실제 코드를 컨텍스트로 공급**하는 것이 핵심이다.

**패턴 1: 멀티모듈 의존 관계 파악**

```bash
# ✅ 실제 빌드 파일로 컨텍스트 공급
@settings.gradle
@core/build.gradle
@api/build.gradle
@infra/build.gradle
이 멀티모듈 구조에서 의존 방향과 순환 의존 위험 지점을 분석해줘
```

**패턴 2: 레이어드 아키텍처 전체 흐름 추적**

```bash
@src/main/java/com/example/api/controller/OrderController.java
@src/main/java/com/example/service/OrderService.java
@src/main/java/com/example/domain/Order.java
@src/main/java/com/example/infra/repository/OrderJpaRepository.java

주문 생성 요청이 Controller → Service → Repository → DB까지
흐르는 과정에서 각 레이어의 책임 경계가 올바른지 분석해줘.
도메인 로직이 Service와 Entity 중 어디에 있어야 하는지 판단해줘.
```

**패턴 3: 트랜잭션 경계 분석**

```bash
@src/main/java/com/example/service/OrderService.java
@src/main/java/com/example/service/PaymentService.java
@src/main/java/com/example/service/InventoryService.java

주문 → 결제 → 재고 차감 흐름에서 각 서비스의 @Transactional 설정이
올바른지, 실패 시 롤백 범위가 의도대로 동작하는지 분석해줘.
트랜잭션 전파(Propagation) 설정 추천도 포함해줘.
```

**패턴 4: 테스트 코드로 동작 역추론**

```bash
# 코드가 복잡할 때, 테스트가 명세서 역할을 한다
@src/test/java/com/example/service/OrderServiceTest.java

이 테스트 코드를 기반으로 OrderService가 구현하는 비즈니스 규칙을
역으로 추론해서 설명해줘. 테스트가 검증하지 않는 엣지 케이스도 도출해줘.
```

**패턴 5: 점진적 컨텍스트 확장 (토큰 절약 + 품질 유지)**

한 번에 모든 파일을 던지면 토큰 낭비 + 품질 저하다. 좁게 시작해서 필요 시 확장한다.

```bash
# Step 1: 핵심 도메인 먼저
@src/main/java/com/example/domain/Order.java
Order 엔티티의 상태 전이 규칙을 파악해줘

# Step 2: 이해 확인 후 Service 추가
@src/main/java/com/example/service/OrderService.java
위 상태 전이를 Service에서 올바르게 제어하고 있는지 확인해줘

# Step 3: 문제 발견 시 관련 파일 추가
@src/main/java/com/example/infra/event/OrderEventPublisher.java
상태 변경 시 이벤트 발행 타이밍이 트랜잭션 커밋 전후 어느 시점인지 확인해줘
```

---

### 7. 커스텀 슬래시 커맨드

반복적인 작업(코드 리뷰, PR 작성, 보안 검토 등)을 프롬프트로 작성해두고 명령어로 실행한다.

```bash
# 프로젝트 전용 커맨드
mkdir -p .claude/commands

# 전역 커맨드 (모든 프로젝트에서 사용)
mkdir -p ~/.claude/commands
```

마크다운 파일을 생성하면 파일명이 커맨드명이 된다.

```bash
cat > .claude/commands/jpa-review.md << 'EOF'
# /jpa-review
@$ARGUMENTS JPA 엔티티를 다음 관점에서 검토한다:
1. N+1 문제 발생 가능 지점 (FetchType.LAZY 확인)
2. 양방향 연관관계 편의 메서드 누락 여부
3. equals/hashCode 구현 방식 (id 기반 권장)
4. 영속성 전이(CascadeType) 남용 여부
각 항목을 ✅/⚠️/❌ 로 명확히 구분해 출력한다.
EOF
```

```bash
# 사용
/jpa-review src/main/java/com/example/domain/Order.java
```

인자가 필요한 경우 `$ARGUMENTS`로 입력받는다.

**Spring 백엔드 실전 커맨드 예시**

```
.claude/commands/
├── jpa-review.md        # JPA 엔티티 검토
├── layer-check.md       # 레이어 의존 방향 검증
├── tx-audit.md          # 트랜잭션 경계 감사
└── pr-draft.md          # PR 설명 자동 초안 작성
```

---

### 8. 서브에이전트 (Subagents)

하나의 세션에서 "리팩터링 전문가", "보안 전문가", "코드 리뷰어"를 동시에 두면 컨텍스트가 오염된다. 서브에이전트는 각자 **독립된 컨텍스트 창**을 가지므로 전문 영역에 집중할 수 있다.

**서브에이전트 특징**

- 특정 목적과 전문 영역을 가진 사전 구성된 AI 인격체
- 기본 대화와 별도로 자체 컨텍스트 창을 사용
- 허용된 특정 도구로 구성 가능
- 사용자 정의 시스템 프롬프트 보유

**직접 생성**

```bash
/agents
# → "Create New Agent" 선택
# → 에이전트 역할 / 전문 영역 / 도구 권한 설정
# → "e" 키로 시스템 프롬프트 직접 편집
```

**사용**

```bash
# Claude Code가 전문 영역과 일치하는 작업 감지 시 자동 위임
# 또는 명시적 호출
"Use the code-reviewer subagent to check my recent changes"
"Use the jpa-specialist subagent to review Order entity"
```

**트레이드오프**

| 구분 | 직접 요청 | 서브에이전트 |
|---|---|---|
| 설정 비용 | 없음 | 초기 설정 필요 |
| 컨텍스트 | 메인 세션과 공유 | 독립 컨텍스트 |
| 전문성 | 일반 | 특화 영역 집중 |
| 적합한 경우 | 단순 작업 | 복잡한 전문 검토 |

> 매번 직접 만들기 번거롭다면 [Super Claude](https://github.com/NickSavage/super-claude)처럼 잘 만들어진 서브에이전트 모음을 활용하는 것도 좋은 선택이다.

---

### 9. 개인 설정 — 편의성 증대

#### 9-1. `~/.claude/CLAUDE.md` — 전역 개인 설정

모든 프로젝트에 공통 적용된다.

```markdown
# 개인 전역 설정

## 응답 스타일
- 코드 수정 시 변경 이유를 먼저 설명하고 코드를 제시할 것
- 트레이드오프가 있는 경우 반드시 명시할 것
- 한국어로 응답할 것

## 코드 컨벤션 (모든 Java 프로젝트 공통)
- Java 17 이상, record 클래스 적극 활용
- var 사용 지양 (명시적 타입 선호)
- 메서드 길이 20줄 초과 시 분리 제안할 것
- 코드 제안 시 테스트 가능성(Testability) 관점 포함
```

#### 9-2. `settings.json` — 권한 제어 및 Git 귀속 설정

기본적으로 Claude Code는 Git 커밋과 PR에 `Co-Authored-By: Claude` 문구를 자동으로 추가한다. 팀 컨벤션에 맞게 제거하거나 수정하려면 `attribution` 필드를 설정한다.

```json
// ~/.claude/settings.json (전역 적용)
{
  "model": "claude-sonnet-4-20250514",
  "permissions": {
    "allowedTools": ["Read", "Write", "Bash(git *)", "Bash(./gradlew *)"],
    "deny": [
      "Read(**/.env*)",
      "Write(**/application-prod*)"
    ]
  },
  "attribution": {
    "commit": "",
    "pr": ""
  }
}
```

#### 9-3. Plan 모드 — 코드 수정 전 설계 검토

`Shift+Tab`을 누를 때마다 일반 모드 → 자동 승인 모드 → 플랜 모드를 순환한다.

```bash
# Shift+Tab → Plan mode 활성화
"OrderService에 환불 로직을 추가하려고 해.
어떤 파일을 어떻게 수정할지 계획만 먼저 제시해줘"

# 계획 검토 후 Shift+Tab → Normal mode로 전환해서 실제 구현
```

영향 범위가 넓은 Spring 설계 작업에서 유용하다. 코드가 바뀌기 전에 방향을 먼저 검토할 수 있다.

#### 9-4. Hooks — 반복 작업 자동화

Hooks는 Claude의 판단과 무관하게 항상 결정론적으로 실행되는 라이프사이클 자동화 기능이다.

```json
// .claude/settings.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write(*.java)",
        "hooks": [
          { "type": "command", "command": "./gradlew :spotlessApply --quiet" }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash(git commit*)",
        "hooks": [
          { "type": "command", "command": "./gradlew test --quiet" }
        ]
      }
    ]
  }
}
```

Java 파일 수정 시 Spotless 자동 실행 + `git commit` 전 테스트 자동 실행.

#### 9-5. Mac OS 시스템 알림

Claude Code는 비동기로 작업을 진행하기 때문에 완료 시점을 알기 어렵다. `claude-notify`로 완료 알림을 받을 수 있다.

```bash
pip install claude-notify
```

```json
// .claude/settings.json에 추가
{
  "hooks": {
    "Notification": [
      {
        "matcher": ".*",
        "hooks": [{ "type": "command", "command": "claude-notify hook" }]
      }
    ],
    "Stop": [
      {
        "matcher": ".*",
        "hooks": [{ "type": "command", "command": "claude-notify hook" }]
      }
    ]
  }
}
```

작업이 끝나면 Mac OS 시스템 알림이 와서 다음 작업을 바로 이어갈 수 있다.

---

### 10. Notion MCP 연동 — 도메인별 기능 문서화

**왜 Notion인가?**

Notion은 공식 MCP 서버를 직접 구축해 유지관리하고 있다. Google Docs는 아직 공식 MCP 서버가 없다. Notion의 호스팅 MCP 서버는 기존 API가 반환하던 계층적 JSON 블록 데이터를 마크다운으로 변환해 AI 에이전트에 최적화된 응답을 제공한다. 결과적으로 토큰 효율이 높다.

**MCP(Model Context Protocol)란?**

Claude Code가 로컬 파일 시스템 밖의 외부 서비스와 통신하기 위한 표준 프로토콜이다. Anthropic이 오픈소스로 만들었고 OpenAI, Google 등도 채택했다. MCP 서버가 연결되면 Claude Code는 Notion을 읽고 쓸 수 있는 손이 생기는 것이다.

#### 설치 — 방법 A: 공식 호스팅 서버 (권장)

```bash
# OAuth 인증 기반, 토큰 관리 자동
claude mcp add --transport http notion https://mcp.notion.com/mcp

# 세션 내에서 인증
/mcp
# → 브라우저 열림 → Notion 계정 로그인 → 워크스페이스 승인
```

#### 설치 — 방법 B: npm 패키지 (직접 제어)

```bash
# 1. https://www.notion.so/profile/integrations → 새 통합 생성 → 토큰 복사

# 2. ~/.claude/settings.json 에 등록
{
  "mcpServers": {
    "notionApi": {
      "command": "npx",
      "args": ["-y", "@notionhq/notion-mcp-server"],
      "env": { "NOTION_TOKEN": "ntn_xxxx" }
    }
  }
}
```

> **주의**: 방법 B 사용 시, Claude가 접근할 Notion 페이지에 Integration을 반드시 수동으로 연결해야 한다. 페이지의 `•••` 메뉴 → `Add connections` → 생성한 통합 선택. 이 단계를 빠뜨리면 유효한 토큰이 있어도 `object not found` 에러가 발생한다.

**설치 확인**

```bash
/mcp       # 연결된 MCP 서버 목록 확인
/context   # Notion MCP 토큰 소비량 확인
```

**방법 A vs B**

| | 방법 A (OAuth 호스팅) | 방법 B (npm 패키지) |
|---|---|---|
| 설정 난이도 | 쉬움 | 보통 |
| 토큰 관리 | 자동 | 직접 관리 |
| 팀 공유 | `.mcp.json`으로 공유 가능 | 각자 토큰 필요 |
| 적합한 경우 | 개인/소규모 팀 | 세밀한 접근 제어 필요 시 |

#### 도메인별 핵심 기능 문서화 커스텀 커맨드

```markdown
<!-- .claude/commands/doc-domain.md -->

# /doc-domain
$ARGUMENTS 도메인의 핵심 기능을 Notion에 문서화한다.

1. 해당 도메인 관련 파일을 읽어 다음을 분석한다:
   - 도메인 엔티티와 핵심 비즈니스 규칙
   - 주요 API 엔드포인트 및 Request/Response 구조
   - 트랜잭션 경계 및 외부 의존성
   - 주요 예외 케이스 및 처리 방식

2. Notion의 "도메인 설계 문서" 페이지 하위에 다음 구조로 페이지를 생성한다:
   ## 도메인 개요
   ## 엔티티 구조 및 상태 전이
   ## 핵심 API 명세
   ## 트랜잭션 전략
   ## 알려진 제약사항 및 예외 처리

3. 생성된 Notion 페이지 URL을 출력한다.
```

**실제 사용 흐름**

```bash
# 주문 도메인 전체 문서화
/doc-domain Order

# 또는 파일을 직접 지정해서 요청
"@src/main/java/com/example/api/controller/PaymentController.java
@src/main/java/com/example/service/PaymentService.java
@src/main/java/com/example/domain/Payment.java
결제 도메인의 핵심 기능을 Notion '백엔드 도메인 설계' 페이지 하위에 문서화해줘"
```

**Notion에서 자동 생성되는 문서 구조**

```
📚 백엔드 도메인 설계
  ├── Order 도메인 (2026-04-07 업데이트)
  │     ├── 📌 도메인 개요
  │     ├── 🔄 엔티티 구조 및 상태 전이
  │     ├── 🌐 핵심 API 명세
  │     ├── 💾 트랜잭션 전략
  │     └── ⚠️ 제약사항 및 예외 처리
  └── Payment 도메인
        └── ...
```

---

### 11. 커뮤니티 꿀팁 모음

**① 세션 재시작 없이 이전 대화 이어가기**

```bash
claude -c   # 마지막 세션 대화에서 이어서 시작
```

**② 터미널에서 바로 일회성 질의**

```bash
claude -p "이 에러 원인 분석해줘" < error.log
claude -p "summarize this file" < README.md
```

**③ Plan mode로 대규모 작업 전 영향 범위 먼저 파악**

코드가 바뀌기 전에 Claude가 어떤 파일을 어떻게 수정할지 계획만 먼저 보여준다. 방향이 틀렸다면 이 단계에서 잡으면 된다.

**④ CLAUDE.md 결정 로그 패턴 — 컨텍스트 유지의 핵심**

```markdown
## [설계 결정 로그]
- 2026-04-07: Saga 패턴 채택 (2PC 대비 서비스 독립성 우선)
- 2026-04-07: 낙관적 락(@Version) 전체 엔티티에 적용
```

세션이 끊겨도 CLAUDE.md가 자동 주입되므로 결정 맥락이 살아있다.

**⑤ 서브에이전트로 코드 리뷰 분리**

코드 리뷰를 메인 세션에서 하면 컨텍스트가 오염된다. 전용 서브에이전트에게 위임하면 메인 흐름에 영향을 주지 않는다.

```bash
"Use the code-reviewer subagent to review the Order domain changes"
```

**⑥ `/context`로 토큰 낭비 주범 찾기**

```bash
/context
# MCP 서버별 토큰 소비량 확인 → 불필요한 MCP 비활성화 판단
```

**⑦ 자동 메모리 비활성화 (민감한 프로젝트)**

```json
// .claude/settings.json
{
  "autoMemoryEnabled": false
}
```

---

## 💻 코드 실습 및 분석

### 실습 1: CLAUDE.md 초기 세팅

```bash
# 1. 프로젝트 루트에서 시작
claude

# 2. 프로젝트 자동 분석 후 CLAUDE.md 초안 생성
/init

# 3. # 단축 문법으로 팀 컨벤션 즉시 추가
# 트랜잭션은 Service 레이어에서만 선언
# 엔티티 패키지: com.example.core.domain

# 4. 결정 로그 섹션 추가
# [설계 결정 로그]
# - 2026-04-07: 멀티모듈 구조 확정 (core/api/infra)
```

### 실습 2: /compact 후 컨텍스트 재주입 패턴

```bash
# 상황: OrderService 리팩터링 대화가 길어짐

# 1. 압축 실행
/compact

# 2. 결정 사항 명시적 재주입 (반드시)
"지금까지 결정된 사항:
 - Order.create() 정적 팩토리 메서드 완성됨
 - OrderService는 팩토리 호출 + 이벤트 발행 담당
 - Payment 연동은 Saga 패턴으로 처리 예정
 다음으로 PaymentService 구현 시작해줘"
```

### 실습 3: 모델 전환 + 셸 조합 워크플로우

```bash
# 1. 복잡한 설계 → Opus
/model opus
"@src/main/java/com/example/service/OrderService.java
@src/main/java/com/example/service/PaymentService.java
두 서비스 간 트랜잭션 전략으로 Saga vs 2PC 트레이드오프 분석해줘"

# 2. 설계 확정 후 구현 → Sonnet으로 내리기
/model sonnet
"방금 논의한 Saga 패턴으로 OrderService 구현해줘"

# 3. 구현 후 테스트 (토큰 절약)
!./gradlew test

# 4. 실패 시 바로 분석 요청
"위 테스트 실패 원인 분석하고 수정해줘"
```

### 실습 4: JPA 리뷰 커스텀 커맨드 생성

```bash
mkdir -p .claude/commands

cat > .claude/commands/jpa-review.md << 'EOF'
# /jpa-review
@$ARGUMENTS JPA 엔티티를 다음 관점에서 검토한다:
1. N+1 문제 발생 가능 지점 (FetchType.LAZY 확인)
2. 양방향 연관관계 편의 메서드 누락 여부
3. equals/hashCode 구현 방식 (id 기반 권장)
4. 영속성 전이(CascadeType) 남용 여부
각 항목을 ✅/⚠️/❌ 로 명확히 구분해 출력한다.
EOF

# 사용
/jpa-review src/main/java/com/example/domain/Order.java
```

---

## 💡 회고 및 질의응답

**Q. `/compact`와 `/clear`는 언제 각각 써야 하나?**

`/compact`는 같은 태스크 맥락을 유지하면서 토큰이 부족할 때 쓴다. 단, 압축 후 반드시 핵심 결정 사항을 수동으로 재주입해야 맥락이 유지된다. `/clear`는 완전히 다른 태스크로 전환할 때 쓴다. 이전 맥락이 오히려 방해가 되는 경우다.

**Q. GPT보다 맥락 유지가 안 되는 느낌이 드는 이유는?**

Claude Code는 대화 기억보다 파일(CLAUDE.md)을 통한 상태 관리가 기본 철학이다. GPT가 대화 흐름에 의존하는 방식이라면, Claude Code는 파일 시스템 에이전트로 설계됐기 때문에 CLAUDE.md에 결정 로그를 남기는 습관을 들이면 세션이 끊겨도 맥락이 살아있다.

**Q. 커스텀 커맨드와 서브에이전트는 언제 각각 쓰나?**

커스텀 커맨드는 내가 명시적으로 호출하는 반복 작업에 쓴다 (`/jpa-review`, `/pr-draft`). 서브에이전트는 특정 영역에 전문성이 필요하고 독립 컨텍스트가 필요할 때 쓴다 (보안 전문가, 리팩터링 전문가).

**Q. Notion MCP 연동 시 모든 페이지에 접근 가능한가?**

방법 B(npm 패키지) 사용 시 각 페이지에 Integration을 수동 연결해야 접근 가능하다. 프로덕션 데이터가 있는 페이지는 접근 범위에서 빼는 것이 안전하다.

**Q. Hooks 실패 시 Claude 작업도 중단되나?**

`PreToolUse` Hook에서 테스트가 실패하면 Claude가 오류를 인지하고 커밋 대신 수정을 제안한다. 결정론적으로 실행되므로 Claude의 판단을 우회해 항상 동작한다.

---

> **참고 공식 문서**
>
> - Claude Code CLI 레퍼런스: <https://code.claude.com/docs/en/cli-reference>
> - Claude Code MCP 연동: <https://code.claude.com/docs/en/mcp>
> - Notion MCP 공식 가이드: <https://developers.notion.com/guides/mcp/get-started-with-mcp>
> - Notion 공식 Claude Code 플러그인: <https://github.com/makenotion/claude-code-notion-plugin>
