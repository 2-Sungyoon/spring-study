# 📅 [week 8] 주제: Claude Code 실전 활용 — IntelliJ 연동 & 워크플로우

## 🎯 학습 목표

- CLAUDE.md 기반 메모리 시스템으로 세션 간 컨텍스트를 유지하는 방법을 익힌다
- 커스텀 슬래시 커맨드, 서브에이전트, Skills를 활용해 반복 작업을 자동화한다
- `/compact` 이후에도 맥락이 끊기는 근본 원인을 이해하고 대응 전략을 세운다
- 2026년 신규 기능 (`/loop`, `/branch`, `/voice`, Remote Control)을 파악한다
- Java/Spring 프로젝트에 맞는 워크플로우 패턴을 수립한다

---

## 📝 주요 개념 정리

### 1. Claude Code 포지셔닝

Claude Code는 CLI 기반 에이전틱 코딩 어시스턴트다. 브라우저 Claude와 근본적으로 다른 점은 파일을 읽고 수정하고 셸 명령을 실행하는 **에이전트로서 작동**한다는 것이다.

| | Claude Code | GitHub Copilot | Cursor |
|---|---|---|---|
| 컨텍스트 범위 | 프로젝트 전체 | 현재 파일 중심 | 프로젝트 전체 |
| 에이전틱 실행 | ✅ 파일 수정, 셸 실행 | ❌ | 제한적 |
| 커스터마이징 | CLAUDE.md, Skills, Hooks | 제한적 | Rules 파일 |
| IDE 종속 | CLI 기반 (IDE 무관) | IDE 플러그인 | 자체 IDE 교체 필요 |

---

### 2. 커맨드 체계와 실전 단축 문법

**핵심 슬래시 커맨드**

```
/help        현재 세션에서 사용 가능한 모든 커맨드 확인 (커스텀 포함)
/init        프로젝트 스캔 후 CLAUDE.md 자동 생성
/model       모델 전환 (복잡한 설계 → Opus, 단순 작업 → Haiku)
/compact     컨텍스트 압축 (토큰 부족 시)
/clear       대화 초기화 (완전히 다른 태스크 전환 시)
/context     현재 세션 토큰 사용량 확인 (MCP 서버별 포함)
/rewind      특정 체크포인트로 대화/코드 상태 되돌리기
/mcp         연결된 MCP 서버 목록 및 관리
/agents      서브에이전트 관리 (생성/선택/삭제)
```

**실전 단축 문법 3가지**

`@파일참조` — 특정 파일을 컨텍스트로 포함

```bash
@src/main/java/com/example/service/OrderService.java 이 클래스의 의존성을 분석해줘
```

`!셸실행` — 대화형 모드 우회 (토큰 소비 없이 직접 실행)

```bash
!./gradlew test
!git diff HEAD
```

`#메모리` — CLAUDE.md에 즉시 저장

```bash
# 트랜잭션은 Service 레이어에서만 선언, @Transactional(readOnly=true) 기본값
```

---

### 3. CLAUDE.md — 프로젝트 메모리

계층 구조로 동작한다.

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

# [설계 결정 로그]
- 2026-04-07: 멀티모듈 구조 확정 (core/api/infra)
```

> 핵심 규칙만 20줄 이내로 유지하고, 상세 문서는 별도 파일로 분리해 `@docs/architecture-rules.md` 형태로 참조하는 것이 낫다.

---

### 4. 컨텍스트 유지 문제 — 근본 원인과 대응

> 💬 *"GPT보다 맥락 유지가 안 되는 것 같다. `/compact` 이후든 같은 프로젝트 내에서든"*

**원인 1: `/compact`는 요약이지 기억이 아니다**

`/compact`는 대화를 압축 요약한다. 세부 결정 사항과 논의 맥락이 손실되므로, 압축 직후 핵심 결정 사항을 수동으로 재주입해야 한다.

```bash
# ❌ /compact만 믿고 이어가기
/compact
"계속해줘"  # Claude는 무엇을 기준으로 계속해야 하는지 모름

# ✅ /compact 직후 결정 사항 명시적 재공급
/compact
"지금까지 결정된 사항:
1. Order 엔티티에 정적 팩토리 메서드 도입
2. Service는 팩토리 호출만 담당, 비즈니스 로직은 엔티티에
이 방향으로 PaymentService 구현 계속해줘"
```

**원인 2: 세션 간 컨텍스트는 자동으로 이어지지 않는다**

CLAUDE.md에 의사결정 로그를 남기면 세션이 끊겨도 매 세션마다 자동 주입되어 맥락이 유지된다.

```markdown
## [설계 결정 로그]
- 2026-04-07: Saga 패턴 채택 (분산 트랜잭션 대신, 서비스 독립성 우선)
- 2026-04-07: Order 엔티티 정적 팩토리 메서드 도입 (Order.create(...))
```

**원인 3: 컨텍스트 오염 (Context Rot)**

대화가 길어질수록 초반 제약조건이 희석된다.

```
같은 태스크 맥락 유지 + 토큰 부족   → /compact + 결정 사항 재주입
완전히 다른 태스크로 전환            → /clear
세션 재시작                          → CLAUDE.md 결정 로그가 자동 주입됨
```

**GPT vs Claude Code 근본적 차이**

GPT는 대화 흐름에 의존하지만, Claude Code는 **파일을 통한 상태 관리**가 기본 철학이다. CLAUDE.md에 결정 로그를 남기는 습관이 맥락 유지의 핵심이다.

---

### 5. 토큰 절약 전략

**① 점진적 컨텍스트 확장 — 한 번에 다 던지지 않는다**

```bash
# Step 1: 핵심 도메인 먼저
@src/main/java/com/example/domain/Order.java
Order 엔티티의 상태 전이 규칙을 파악해줘

# Step 2: 이해 확인 후 Service 추가
@src/main/java/com/example/service/OrderService.java
위 상태 전이를 Service에서 올바르게 제어하고 있는지 확인해줘
```

**② 셸 명령은 `!`로 직접 실행**

```bash
!./gradlew test   # 대화형 모드 우회 → 토큰 없이 바로 실행
```

**③ 토큰 현황 파악**

```bash
/context   # MCP 서버별 소비량 포함해서 확인
```

**④ 모델 전략적 전환**

```bash
/model haiku   # 보일러플레이트, 단순 리팩터링
/model sonnet  # 일반 기능 구현, 버그 수정 (기본값)
/model opus    # 아키텍처 설계, 복잡한 트랜잭션 로직 분석
```

---

### 6. 복잡한 Spring 코드 구조 이해시키기

**패턴 1: 멀티모듈 의존 관계 파악**

```bash
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
```

**패턴 4: 테스트 코드로 동작 역추론**

```bash
@src/test/java/com/example/service/OrderServiceTest.java

이 테스트 코드를 기반으로 OrderService가 구현하는 비즈니스 규칙을
역으로 추론해서 설명해줘. 테스트가 검증하지 않는 엣지 케이스도 도출해줘.
```

---

### 7. 커스텀 슬래시 커맨드

반복적인 작업을 프롬프트로 작성해두고 명령어로 실행한다. 마크다운 파일을 생성하면 파일명이 커맨드명이 된다.

```bash
mkdir -p .claude/commands        # 프로젝트 전용
mkdir -p ~/.claude/commands      # 전역 (모든 프로젝트)
```

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

# 사용
/jpa-review src/main/java/com/example/domain/Order.java
```

> 현재 Claude Code의 권장 방식은 `.claude/commands/` 대신 `.claude/skills/`다. 기존 commands 파일도 그대로 동작하지만, 신규 작성은 Skills 형식을 쓰는 것이 낫다. (7절 Skills 참고)

---

### 8. 서브에이전트 (Subagents)

하나의 세션에서 "리팩터링 전문가"와 "보안 전문가"를 동시에 두면 컨텍스트가 오염된다. 서브에이전트는 **독립된 컨텍스트 창**을 가지므로 전문 영역에 집중할 수 있다.

```bash
/agents
# → "Create New Agent" 선택
# → 에이전트 역할 / 전문 영역 / 도구 권한 설정
# → "e" 키로 시스템 프롬프트 직접 편집
```

```bash
# 명시적 호출
"Use the jpa-specialist subagent to review Order entity"
```

| 구분 | 직접 요청 | 서브에이전트 |
|---|---|---|
| 설정 비용 | 없음 | 초기 설정 필요 |
| 컨텍스트 | 메인 세션과 공유 | 독립 컨텍스트 |
| 전문성 | 일반 | 특화 영역 집중 |

> [Super Claude](https://github.com/NickSavage/super-claude)처럼 잘 만들어진 서브에이전트 모음을 활용하는 것도 좋은 선택이다.

---

### 9. Skills — 커스텀 커맨드의 진화형

커스텀 커맨드(`.claude/commands/`)와 스킬(`.claude/skills/`)이 통합됐다. 기존 commands 파일은 그대로 동작하지만, Skills는 여기에 더해 프론트매터 제어, 지원 파일 번들링, Claude 자동 호출 등을 지원한다.

```
.claude/
  skills/
    jpa-review/
      SKILL.md       ← 메인 명령 정의
      checklist.md   ← 지원 파일 (체크리스트, 예시 등 번들 가능)
```

**Skills 프론트매터 예시**

```markdown
---
name: jpa-review
description: JPA 엔티티를 N+1, 연관관계, 락 전략 관점에서 검토한다
allowed-tools: Read, Bash(./gradlew *)
context: fork
---

@$ARGUMENTS 엔티티를 다음 관점에서 검토한다:
1. N+1 문제 발생 가능 지점
2. 양방향 연관관계 편의 메서드 누락 여부
3. @Version 낙관적 락 필요성
```

- `context: fork` — 독립 서브에이전트에서 실행 → 메인 컨텍스트 오염 없음
- `allowed-tools` — 해당 스킬에서만 허용할 도구 제한
- 파일명과 동일하게 `/jpa-review`로 호출

**빌트인 Skills (설치 없이 바로 사용 가능)**

```
/simplify    변경된 파일을 3개 병렬 에이전트가 검토 → 아키텍처 이슈, 중복 로직, 성능 개선점 자동 수정
/review      코드 리뷰 수행
/batch       여러 파일에 동일한 변경을 병렬로 적용 (패턴 변경, 마이그레이션에 유용)
/debug       디버깅 지원
/claude-api  Anthropic SDK 관련 문서, 패턴, 예시 제공
```

**코드 설명 스킬 (직접 정의)**

Claude Code가 복잡한 코드를 특정 독자 수준에 맞춰 설명하게 만드는 커스텀 스킬이다.

```markdown
---
name: explain-code
description: 코드를 독자 수준에 맞게 설명한다. 팀 온보딩이나 PR 리뷰 시 유용.
---

@$ARGUMENTS 코드를 다음 순서로 설명한다:
1. 이 코드가 존재하는 이유 (What & Why)
2. 전체 동작 흐름 (How) — 순차적으로
3. 주요 설계 결정과 트레이드오프
4. 주의해야 할 엣지 케이스
독자는 Spring을 처음 접하는 백엔드 신규 팀원으로 가정한다.
```

```bash
# 사용
/explain-code src/main/java/com/example/service/PaymentService.java
```

**딥 리서치 스킬 (커뮤니티 / 직접 정의)**

라이브러리 도입, 기술 비교, 아키텍처 결정 전 체계적인 조사가 필요할 때 쓴다.

```markdown
---
name: deep-research
description: 기술 주제를 구조적으로 조사하고 의사결정에 쓸 수 있는 보고서를 만든다
context: fork
agent: Explore
---

$ARGUMENTS 주제를 다음 순서로 조사한다:
1. 관련 파일과 코드를 Glob, Grep으로 탐색
2. 공식 문서와 레퍼런스 분석
3. 대안 기술과의 비교 (트레이드오프 포함)
4. 현재 프로젝트 컨텍스트에 맞는 도입 권고안 제시
결과는 요약본을 먼저, 상세 내용은 섹션으로 구분해 출력한다.
```

```bash
# 사용 예시
/deep-research "JPA 낙관적 락 vs 비관적 락 — 쇼핑몰 주문 도메인 기준"
/deep-research "Redis 분산 락 도입 필요성 — 재고 차감 동시성 문제"
```

---

### 10. 2026 신규 기능 — 핵심 커맨드 4가지

#### `/btw` — 메인 흐름 오염 없는 사이드 질문

현재 작업 컨텍스트에 영향을 주지 않고 단발성 질문을 던진다. 구현 중 모르는 개념이 생겼을 때 사용한다.

```bash
# OrderService 구현 중 트랜잭션 전파 방식이 헷갈릴 때
/btw "REQUIRES_NEW와 NESTED의 차이가 뭐야?"

# 답변이 돌아오지만 메인 대화 컨텍스트에 끼어들지 않음
```

> 메인 세션에서 직접 물어보면 컨텍스트가 오염된다. `/btw`를 쓰면 현재 작업 흐름을 유지한 채 빠른 참조가 가능하다.

#### `/loop` — 반복 실행 스케줄링

지정한 간격마다 특정 프롬프트를 반복 실행한다. CI 결과 모니터링, 배포 상태 확인 등 반복 확인 작업에 유용하다.

```bash
# 문법: /loop <간격> <프롬프트>

/loop 5m "!./gradlew test 결과를 확인하고 실패한 테스트가 있으면 보고해줘"
/loop 10m "git log --oneline -5 를 확인하고 새 커밋이 있으면 변경사항 요약해줘"
```

> 장시간 빌드나 배포 파이프라인 모니터링에 유용하다. 멈추려면 `Ctrl+C`.

#### `/branch` — 대화 분기 (실험적 접근에 유용)

현재 대화 상태를 분기해 새 세션으로 복사한다. v2.1.77 이전에는 `/fork`였다. git의 브랜치 개념과 동일하다.

```bash
# 현재 대화 상태에서 분기
/branch

# 활용 패턴
# 1. 복잡한 리팩터링 방향 A와 B를 각각 브랜치에서 시도
# 2. 원하지 않으면 해당 브랜치를 버리고 메인으로 복귀
# 3. /rewind와 달리 메인 대화를 건드리지 않음
```

```bash
# 예시
"OrderService 리팩터링을 도메인 이벤트 방식으로 시도해볼게"
/branch   # 현재 상태 분기
"도메인 이벤트 방식으로 OrderService 구현해줘"
# 결과가 마음에 안 들면 이 브랜치만 버림
```

#### `/voice` — 음성 입력 (2026년 3월 출시, 점진적 롤아웃)

Push-to-Talk 방식. 스페이스바를 누르고 있는 동안 말하고, 놓으면 전송한다. 현재 20개 언어 지원.

```bash
/voice   # 토글 on/off

# 키 커스터마이징 가능 (keybindings.json)
# { "voice:pushToTalk": "meta+k" }
```

> 복잡한 요구사항을 타이핑하는 것보다 말로 설명하는 게 빠를 때 유용하다. 현재 전체 롤아웃 진행 중이라 환영 화면에 접근 가능 여부가 표시된다.

#### Remote Control — 로컬 세션을 외부에서 제어

2026년 2월 리서치 프리뷰로 출시됐다. 터미널에서 Claude Code를 실행한 채로 `claude.ai/code` 웹 인터페이스나 iOS/Android 앱에서 같은 세션에 접속할 수 있다.

```bash
# 세션 시작 시 활성화
claude --remote-control

# 또는 설정에서 항상 활성화
/config   # → Enable Remote Control for all sessions: true
```

> 핵심 보안 설계: 코드는 로컬 머신에 그대로 있다. 채팅 메시지만 암호화 채널로 전송되므로 파일, MCP 서버, 환경변수가 외부로 나가지 않는다.

```
장점: 자리를 비운 사이에도 폰으로 작업 지시 가능
한계: 터미널을 닫으면 세션 종료 / Pro, Max 플랜만 지원 (현재)
```

---

### 11. 개인 설정 — 편의성 증대

#### 11-1. `~/.claude/CLAUDE.md` — 전역 개인 설정

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
```

#### 11-2. `settings.json` — 권한 제어 및 Git 귀속 설정

기본적으로 Claude Code는 Git 커밋에 `Co-Authored-By: Claude` 문구를 자동으로 추가한다.

```json
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

#### 11-3. Plan 모드

`Shift+Tab`을 누를 때마다 일반 모드 → 자동 승인 모드 → 플랜 모드를 순환한다. 코드가 바뀌기 전에 어떤 파일을 어떻게 수정할지 계획만 먼저 확인할 수 있다. 영향 범위가 넓은 Spring 리팩터링 전에 쓰면 잘못된 방향을 사전에 잡을 수 있다.

#### 11-4. Hooks — 반복 작업 자동화

Claude의 판단과 무관하게 항상 결정론적으로 실행되는 라이프사이클 자동화다.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write(*.java)",
        "hooks": [{ "type": "command", "command": "./gradlew :spotlessApply --quiet" }]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash(git commit*)",
        "hooks": [{ "type": "command", "command": "./gradlew test --quiet" }]
      }
    ]
  }
}
```

Java 파일 수정 시 Spotless 자동 실행 + `git commit` 전 테스트 자동 실행.

#### 11-5. Mac OS 시스템 알림

Claude Code는 비동기로 작업을 진행하기 때문에 완료 시점을 알기 어렵다.

```bash
pip install claude-notify
```

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": ".*",
        "hooks": [{ "type": "command", "command": "claude-notify hook" }]
      }
    ]
  }
}
```

---

### 12. Notion MCP 연동 — 도메인별 기능 문서화

MCP(Model Context Protocol)는 Claude Code가 로컬 파일 시스템 밖의 외부 서비스와 통신하기 위한 표준 프로토콜이다. Notion이 공식 MCP 서버를 제공한다.

**설치 — 방법 A: 공식 호스팅 서버 (권장)**

```bash
claude mcp add --transport http notion https://mcp.notion.com/mcp

# 세션 내에서 OAuth 인증
/mcp
```

**설치 — 방법 B: npm 패키지 (직접 제어)**

```json
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

> **주의 (방법 B)**: Claude가 접근할 Notion 페이지에 `•••` → `Add connections`로 Integration을 수동 연결해야 한다. 빠뜨리면 `object not found` 에러가 발생한다.

**도메인별 핵심 기능 문서화 커스텀 커맨드**

```markdown
<!-- .claude/commands/doc-domain.md -->
# /doc-domain
$ARGUMENTS 도메인의 핵심 기능을 Notion에 문서화한다.

1. 해당 도메인 관련 파일을 읽어 분석한다:
   - 엔티티와 핵심 비즈니스 규칙
   - 주요 API 엔드포인트 및 Request/Response 구조
   - 트랜잭션 경계 및 외부 의존성
   - 주요 예외 케이스 및 처리 방식

2. Notion의 "도메인 설계 문서" 하위에 다음 구조로 페이지를 생성한다:
   ## 도메인 개요
   ## 엔티티 구조 및 상태 전이
   ## 핵심 API 명세
   ## 트랜잭션 전략
   ## 알려진 제약사항

3. 생성된 Notion 페이지 URL을 출력한다.
```

```bash
/doc-domain Order    # 주문 도메인 전체 문서화
/doc-domain Payment  # 결제 도메인 문서화
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
```

---

## 💻 코드 실습 및 분석

### 실습 1: 모델 전환 + 셸 조합 워크플로우

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

### 실습 2: /loop 활용 — 빌드 모니터링

```bash
# 긴 통합 테스트 실행 중 5분마다 결과 확인
/loop 5m "!./gradlew integrationTest 상태를 확인하고 실패 테스트가 있으면 이유를 분석해줘"

# 다른 작업을 하다가 알림이 오면 확인
```

### 실습 3: /branch 활용 — 두 가지 설계 비교

```bash
# 현재 상태 저장 후 분기
/branch

# 브랜치 A: 도메인 이벤트 방식
"Order 상태 변경을 도메인 이벤트 기반으로 리팩터링해줘"
# 결과 검토 후 마음에 안 들면 이 브랜치 버림

# 메인으로 돌아가 브랜치 B: 직접 호출 방식 시도
```

### 실습 4: Skills + 딥 리서치 활용

```bash
# 기술 도입 결정 전 체계적 조사
/deep-research "Redis Sorted Set vs RDB로 구현하는 쇼핑몰 랭킹 조회 — 성능과 일관성 트레이드오프"

# 조사 결과를 CLAUDE.md 결정 로그에 반영
# [설계 결정 로그]
# - 2026-04-07: Redis Sorted Set 채택 — 조회 O(log N), 실시간 업데이트 요건 충족
```

---

## 💡 회고 및 질의응답

**Q. Skills와 커스텀 커맨드의 차이는?**

내부적으로 통합됐다. 둘 다 `/커맨드명`으로 호출한다. Skills가 권장 방식이며 프론트매터로 도구 제한, 독립 컨텍스트 실행(`context: fork`), 지원 파일 번들링 등을 추가로 지원한다.

**Q. `/branch`와 `/rewind`의 차이는?**

`/rewind`는 메인 대화를 특정 시점으로 되돌린다 (파괴적). `/branch`는 현재 상태를 복사해 새 세션을 만든다 (비파괴적). 실험적 시도에는 `/branch`가 안전하다.

**Q. `/loop`는 토큰을 많이 소비하나?**

간격마다 새 요청이 발생하므로 누적 토큰 소비가 있다. 단순한 모니터링은 `!` 셸 명령으로 처리하고, Claude가 판단해야 하는 경우에만 `/loop`를 쓰는 것이 효율적이다.

**Q. Remote Control을 쓰면 코드가 외부로 나가나?**

나가지 않는다. 코드는 로컬 머신에 그대로 있고, 채팅 메시지만 Anthropic 서버를 통해 암호화 전송된다. 파일, MCP 서버, 환경변수는 로컬에 유지된다.

**Q. `/compact` 후 맥락이 끊기는 것을 완전히 막을 방법은?**

완전히 막을 수는 없다. 압축 직후 결정 사항을 수동 재주입하고, CLAUDE.md에 설계 결정 로그를 지속적으로 관리하는 것이 현재 가장 현실적인 대응이다.

---

> **참고 공식 문서**
>
> - Claude Code 빌트인 커맨드: <https://code.claude.com/docs/en/commands>
> - Claude Code Skills: <https://code.claude.com/docs/en/skills>
> - Claude Code MCP 연동: <https://code.claude.com/docs/en/mcp>
> - Claude Code Remote Control: <https://code.claude.com/docs/en/remote-control>
> - Notion MCP 공식 가이드: <https://developers.notion.com/guides/mcp/get-started-with-mcp>
