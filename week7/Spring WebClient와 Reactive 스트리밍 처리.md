# 📅 [week 7] 주제: Spring WebClient와 Reactive 스트리밍 처리

## 🎯 학습 목표
- WebClient와 RestTemplate의 차이를 이해한다.

- WebFlux의 Reactive 개념(Mono, Flux)을 이해한다.

- WebClient를 활용한 스트리밍 응답 처리 방식(SSE)을 이해한다.

---

## 📝 주요 개념 정리
### WebClient

**`Spring WebFlux에서 제공하는 비동기 Reactive HTTP Client`**

- Spring WebFlux 프레임워크의 일부
- 반응형(reactive)
- 비동기 / Non-blocking 통신에 유리
- 반응형 HTTP 클라이언트
- 요청 보내면 Mono / Flux로 비동기 응답 받음
- Spring에서 권장

<aside>

**WebFlux란 무엇인가?**

**`Spring의 Reactive 웹 프레임워크`**

- 비동기 / Non-blocking 기반
- 이벤트 기반(event-driven) 처리
- Reactive 프로그래밍
</aside>

즉, WebFlux는 큰 프레임워크, WebClient는 그 안의 도구 하나

### RestTemplate

**`Spring에서 제공하는 동기 HTTP Client`**

- 동기 / Blocking
- 요청 보내면 응답이 올 때까지 기다림
- 객체 형태로 응답 반환
- 현재 유지보수 모드로 사용됨

**어떤 상황에서 어떤 Client를 사용하는 것이 좋을까?**

→ 단순한 동기 API 호출에는 RestTemplate,

비동기 처리나 스트리밍, 대량 요청에는 WebClient가 적합
- 

---

## 💻 코드 실습 및 분석
### PhoMate 코드에서 WebClient / WebFlux 사용

**기본타입**

Reactive 프로그래밍에서는 **`Mono / Flux`**를 사용한다.

- `Mono<T>` : 결과가 0개 또는 1개
- `Flux<T>` : 결과가 여러개(스트리밍 텍스트 조각들)
- `ServerSentEvent<String>` : SSE 이벤트 1개
1. **WebClient 생성**

```java
private final WebClient openAiWebClient;
```

1. **WebClient로 OpenAI API 요청** 

```java
openAiWebClient.post()
        .uri("/chat/completions")
        .contentType(MediaType.APPLICATION_JSON)
        .accept(MediaType.TEXT_EVENT_STREAM)
        .bodyValue(...)
        .retrieve()
        .bodyToFlux(String.class)
```

- 응답을 한 번에 받지 않고 스트리밍 형태로 받음
1. **실제 OpenAI 요청 코드**

```java
Flux<String> textFlux =
        openAiWebClient.post()
                .uri("/chat/completions")
                .contentType(MediaType.APPLICATION_JSON)
                .accept(MediaType.TEXT_EVENT_STREAM)
                .bodyValue(Map.of(
                        "model", openAiModel,
                        "stream", true,
                        "messages", List.of(
                                Map.of("role", "system", "content", buildSystemPrompt()),
                                Map.of("role", "user", "content", request.getUserText())
                        )
                ))
                .retrieve()
                .bodyToFlux(String.class);
```

- "stream", true : OpenAI 응답을 스트리밍 방식으로 받음
1. **bodyToFlux()**

```java
.bodyToFlux(String.class)
```

- bodyToMono()  → 응답 하나
- bodyToFlux()  → 스트리밍 응답
1. **Flux 데이터 가공**

```java
.bodyToFlux(String.class)
.doOnNext(raw -> log.debug("[OPENAI RAW 로그가 발생했어요 !!] {}", raw))
.flatMap(raw -> Flux.fromArray(raw.split("\n")))
.map(String::trim)
.filter(line -> !line.isBlank())
.map(this::normalizeOpenAiStreamPayload)
.filter(data -> data != null && !data.isBlank())
.doOnNext(data -> {
    dataLineCount.incrementAndGet();
    if ("[DONE]".equals(data)) doneCount.incrementAndGet();
    log.debug("[OPENAI DATA !!] {}", data);
})
.filter(data -> !data.equals("[DONE]"))
.map(this::readTreeSafely)
.filter(n -> !n.isMissingNode())
.handle((node, sink) -> {
    JsonNode delta = node.path("choices").path(0).path("delta");
    JsonNode contentNode = delta.path("content");

    if (contentNode.isTextual()) {
        String text = contentNode.asText();
        if (text != null && !text.isEmpty()) {
            sink.next(text);
        }
    } else {
        log.debug("[OPENAI DELTA NO-CONTENT] delta={}", delta);
    }
})
.cast(String.class)
.doOnNext(t -> deltaCount.incrementAndGet());
```

- OpenAI 스트리밍 응답을 파싱해서 실제 텍스트(content)만 추출
1. **SSE로 프론트에 스트리밍 전송**

```java
@Override
public Flux<ServerSentEvent<String>> streamText(ChatStreamRequestDTO request) {
```

- SSE 이벤트를 여러개 스트리밍으로 전송

실제 반환 코드

```java
return textFlux
        .map(text -> {
            assistantAcc.append(text);
            return ServerSentEvent.<String>builder().event("delta").data(text).build();
        })
        .concatWith(Mono.just(ServerSentEvent.<String>builder()
                .event("done")
                .data("ok")
                .build()))
        .onErrorResume(e -> {
            log.error("[STREAM ERROR] sessionId={} userMsgId={}", session.getId(), userMsg.getId(), e);
            return Flux.just(
                    ServerSentEvent.<String>builder().event("error").data("stream_failed").build(),
                    ServerSentEvent.<String>builder().event("done").data("ok").build()
            );
        });
```

- OpenAI → 텍스트 조각 수신 → SSE 이벤트 생성 → 프론트에 실시간 전송
1. **bodyToMono()**

```java
Mono<SearchPlan> planMono = openAiWebClient.post()
        .uri("/chat/completions")
        .contentType(MediaType.APPLICATION_JSON)
        .bodyValue(Map.of(
                "model", openAiModel,
                "stream", false,
                "messages", List.of(
                        Map.of("role", "system", "content", buildQueryPlannerPrompt()),
                        Map.of(
                                "role", "user",
                                "content", buildQueryPlannerUserInput(
                                        session.getCurrentSearchQuery(),
                                        request.getUserText()
                                )
                        )
                )
        ))
        .retrieve()
        .bodyToMono(String.class)
        .map(this::extractAssistantContentFromChatCompletionsJson)
        .map(this::parseSearchPlanJsonSafely);
```

- 스트리밍 X
- 한 번에 응답 1개

---

## 💡 회고 및 질의응답
**그렇다면 PhoMate는 WebFlux 기반의 프로젝트일까?**

- 기본적으로는 Spring MVC 기반의 프로젝트
- OpenAI의 SSE와 같은 특정 기능에서만 WebFlux의 일부 기능(WebClient)을 사용한 것!

→ **`Spring MVC 서버 + WebClient (비동기 API 호출)`**

**한줄요약**

> 
> 
> 
> PhoMate는 기본적으로 Spring MVC 서버지만, OpenAI의 스트리밍 응답을 처리하기 위해 WebFlux의 WebClient와 Flux 기능을 함께 사용했다.
>
