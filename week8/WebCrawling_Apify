# 📅 [week 8] 주제: 웹 크롤링 · 스크래핑과 Apify

## 🎯 학습 목표

- 크롤링과 스크래핑의 차이를 명확히 이해한다.
- 상황에 맞는 크롤링 기술(정적 HTML, 브라우저 자동화, 내부 API 역추적)을 선택할 수 있는 판단 기준을 익힌다.
- Apify 플랫폼의 핵심 구성요소(Actor, Crawlee, Storage, Proxy)를 이해한다.
- 안티봇 우회 전략과 SNS 크롤링 시 주의해야 할 법적·기술적 이슈를 정리한다.
- 백엔드 시스템과 Apify를 연동하는 패턴을 이해한다.

---

## 📝 주요 개념 정리

### 1. 크롤링과 스크래핑의 차이

- **웹 크롤링(Web Crawling)** 은 링크를 따라가며 페이지를 발견하는 행위이다. 핵심은 "어떤 페이지가 존재하는가"를 알아내는 것이다.
- **웹 스크래핑(Web Scraping)** 은 특정 페이지에서 원하는 데이터를 뽑아내는 행위이다. 핵심은 "이 페이지에서 무엇을 가져올 것인가"이다.
- 실무에서는 거의 항상 둘을 함께 쓴다. 카테고리 페이지를 크롤링해 상품 URL을 모으고, 상세 페이지에서 가격·옵션을 스크래핑하는 식이다.
- 인접 개념으로 **Web Harvesting**(크롤링+스크래핑+저장 포괄), **Browser Automation**(폼 제출·결제까지 포함), **RPA**(데스크톱 앱까지 포함)가 있다.

### 2. Apify 플랫폼

- 크롤링·스크래핑·브라우저 자동화를 클라우드에서 서버리스로 실행할 수 있게 해주는 풀스택 플랫폼이다.
- 핵심 실행 단위는 **Actor** — Docker 컨테이너로 패키징된 서버리스 프로그램으로, JSON input을 받고 구조화된 output을 반환한다.
- Apify Store에 5,000개 이상의 기성 Actor가 등록되어 있어 직접 만들지 않고도 활용 가능하다.
- 과금 단위는 **Compute Unit(CU)** 이다. 1 CU = 1GB 메모리로 1시간 실행에 해당한다. 메모리 할당이 곧 CPU 할당과 비례하므로, 작업 특성에 맞게 메모리를 조절하는 것이 비용 최적화의 출발점이다.

### 3. Crawlee(라이브러리)와 Apify(플랫폼)의 관계

- **Crawlee** 는 오픈소스 **라이브러리**이고, **Apify** 는 그 코드를 돌릴 수 있는 **클라우드 플랫폼**이다. 같은 회사가 만들었지만 별개의 것이며 독립적으로 사용할 수 있다.
- Spring 생태계에 비유하면 **Crawlee = Spring Boot(프레임워크)**, **Apify = AWS/Heroku(호스팅 플랫폼)** 의 관계이다. CheerioCrawler·PlaywrightCrawler는 Spring의 `RestController`나 `JpaRepository` 같은 Crawlee 내부 클래스라고 보면 된다.
- 4가지 조합이 가능하다.
    1. **Crawlee 직접 사용 + 내 서버 실행**: 무료지만 프록시·스케줄링·모니터링은 직접 구축해야 한다.
    2. **내가 만든 Crawlee 코드를 Apify Actor로 배포**: 운영 인프라가 모두 포함된다. 비용 발생.
    3. **Crawlee 없이 다른 도구(Jsoup, Playwright-Java 등) 사용**: Java 진영에서 흔한 패턴.
    4. **코드 작성 없이 Apify Store의 기성 Actor를 REST API로 호출**: Spring 백엔드에서 가장 현실적인 패턴이다.
- Spring 백엔드에서 Apify를 사용한다는 것은 보통 4번을 의미한다. 이 경우 Spring 코드 어디에도 Crawlee나 CheerioCrawler가 등장하지 않는다. **Apify는 Spring 입장에서 그저 "크롤링 결과를 돌려주는 외부 API 서비스"** 일 뿐이다(카카오 로그인 API나 토스 결제 API를 호출하는 것과 동일한 구조).
- 그럼에도 Crawlee/Crawler 종류를 알아둬야 하는 이유는 ① **Apify Store에서 Actor를 고를 때** 그 Actor가 어떤 엔진 기반인지 알면 성능과 비용을 가늠할 수 있고, ② 언젠가 직접 Actor를 만들어야 할 때 필요하며, ③ 크롤링이 동작하는 원리를 알아야 디버깅이 가능하기 때문이다.

### 4. Actor 구조와 실행 모드

- Actor는 `.actor/actor.json`(메타 설정), `input_schema.json`(입력 스키마), `src/main.js`(진입점), `Dockerfile` 등으로 구성된다.
- 실행 모드는 두 가지다.
    - **Batch 모드**: input → 작업 수행 → output 반환. 대량 크롤링용이다.
    - **Standby 모드**: 퍼블릭 URL에서 웹서버처럼 상시 대기. 실시간 API 서버나 MCP 서버로 활용한다.
- 스토리지는 **Dataset**(구조화된 결과), **Key-Value Store**(쿠키·설정·파일), **Request Queue**(URL 큐)의 3종이다.

### 5. Crawlee와 Crawler 선택 기준

- Crawlee는 Apify가 만든 오픈소스 크롤링 라이브러리로, Actor의 실제 엔진이다.
- **CheerioCrawler**: 정적 HTML, SSR 페이지, API 응답 파싱용. 4GB RAM으로 분당 500페이지 이상 가능하다. 가장 빠르고 저렴하다.
- **PlaywrightCrawler**: JS 렌더링이 필수인 SPA, 동적 콘텐츠용. Cheerio 대비 약 1/10 속도이지만 어쩔 수 없는 경우가 있다. 탐지 우회 측면에서는 Firefox가 Chromium보다 유리하다.
- **PuppeteerCrawler**: Playwright와 유사하며 Chrome에 특화되어 있다.
- 처음에 Cheerio로 시작하고, 안 되면 Playwright로 올리는 전략이 유효하다. Crawlee가 공통 인터페이스를 제공해 전환이 쉽다.

### 6. 안티봇 / 안티스크래핑 대응

- **프록시 전략**: 데이터센터(저렴, 일반 사이트) → 레지덴셜(가정용 IP, 보호 강한 사이트) → 모바일(4G/5G, 최강 보호) 순으로 신뢰도와 비용이 올라간다.
- **핑거프린트 위장**: Crawlee는 기본 설정만으로도 실제 브라우저 수준의 헤더와 TLS 핑거프린트를 자동 적용한다. 부족하면 `puppeteer-extra-plugin-stealth`, `curl-impersonate` 등을 고려한다.
- **Headless 탐지 포인트**: `navigator.webdriver`, WebGL 렌더러, TLS JA3/JA4 핑거프린트, Canvas/Audio 핑거프린트, 마우스 움직임의 통계적 특성 등이 봇 탐지의 주요 단서다.
- **요청 패턴 관리**: 동시성을 낮추고, 랜덤 딜레이를 넣고, 페이지 탐색 순서를 섞고, `maxRequestRetries`를 10 이상으로 설정한다.
- **숨겨진 API 활용**: 브라우저 크롤링 대신 사이트의 내부 API를 역추적해 직접 호출하는 것이 훨씬 효과적인 경우가 많다. 내부 API는 보통 안티봇 보호가 약하다. SNS 크롤링의 핵심 전략이다.

### 7. SNS 크롤링과 법적 고려사항

- 대부분의 SNS는 자동화 접근을 ToS로 금지한다. 공개 데이터만 수집해도 법적 리스크를 인지해야 한다.
- 플랫폼별 난이도: Instagram(매우 높음, 레지덴셜 프록시 필수), Twitter/X(높음), YouTube(공식 API 우선), Google Maps(중간), 이커머스(비교적 낮음).
- **법적 이슈**: 미국은 hiQ vs LinkedIn 판례로 공개 데이터 스크래핑이 CFAA 위반은 아니지만 ToS 위반은 별개다. 한국은 정보통신망법·저작권법·개인정보보호법이 동시에 적용될 수 있다. EU GDPR이 가장 강력하다.
- 안전 영역은 **공개 데이터 + 개인정보 미포함 + 합리적 요청 빈도 + ToS 위반 최소화**의 조합이다.

### 8. 백엔드 연동 패턴

- Actor는 REST API로 실행·모니터링·결과 조회가 모두 가능하다.
- 일반적인 흐름: 백엔드에서 Actor 실행 API 호출 → Apify에서 Actor 실행 → 완료 시 Webhook 콜백 → 백엔드가 Dataset API로 결과 조회.
- **동기 실행(`run-sync`)** 은 짧은 작업(수십 초 이내)에만 적합하다. 대부분은 **비동기(`run`)** 후 webhook으로 처리한다.
- Actor 간 체이닝으로 파이프라인을 만들 수 있다. 예: 상품 URL 수집기 → 상세 페이지 스크래퍼 → 가격 변동 감지기.
- MCP 서버로도 배포 가능해 Claude, GPT 같은 AI 에이전트와 통합할 수 있다.

### 9. 기술 선택 의사결정 흐름

1. 공식 API가 있다 → 무조건 API 우선
2. 정적 HTML로 충분 → CheerioCrawler
3. JS 렌더링 필수 + 내부 API 역추적 가능 → HTTP 클라이언트로 직접 호출
4. JS 렌더링 필수 + 내부 API 못 찾음 → PlaywrightCrawler
5. 안티봇이 강함 → Playwright(Firefox) + 레지덴셜 프록시 + 스텔스 + 인간 행동 시뮬레이션

비용·복잡도는 위에서 아래로 갈수록 10~100배 증가한다. 항상 위에서부터 시도하고 막힐 때만 한 단계씩 내려가는 것이 원칙이다.

---

## 💻 코드 실습 및 분석

> Spring Boot 백엔드가 Apify Actor를 트리거하고, Webhook으로 완료 알림을 받아 결과를 저장하는 흐름을 구현한다. Apify는 외부 API처럼 사용한다.
> 

### 전체 흐름

```
[Spring Boot]
   ↓ ① POST /v2/acts/{actorId}/runs  (Actor 실행 + webhook 등록)
[Apify Cloud]
   ↓ ② Webhook (실행 완료 콜백)
[Spring Boot]
   ↓ ③ GET /v2/datasets/{id}/items  (결과 조회)
[DB 저장]
```

### 1) 설정

```yaml
# application.yml
apify:
  base-url: https://api.apify.com/v2
  token: ${APIFY_TOKEN}
  actor-id: my-username~my-scraper
  webhook-secret: ${APIFY_WEBHOOK_SECRET}

app:
  public-url: ${PUBLIC_URL}   # 로컬 개발 시 ngrok URL
```

### 2) Apify 호출 서비스

```java
@Service
@Slf4j
public class ApifyService {

    private final WebClient webClient;
    private final String actorId;
    private final String publicUrl;

    public ApifyService(
        @Value("${apify.base-url}") String baseUrl,
        @Value("${apify.token}") String token,
        @Value("${apify.actor-id}") String actorId,
        @Value("${app.public-url}") String publicUrl
    ) {
        this.webClient = WebClient.builder()
            .baseUrl(baseUrl)
            .defaultHeader(HttpHeaders.AUTHORIZATION, "Bearer " + token)
            .build();
        this.actorId = actorId;
        this.publicUrl = publicUrl;
    }

    /** Actor 실행 + webhook 등록 */
    public void startRun(List<String> urls) {
        Map<String, Object> input = Map.of(
            "startUrls", urls.stream().map(u -> Map.of("url", u)).toList()
        );

        webClient.post()
            .uri(uriBuilder -> uriBuilder
                .path("/acts/{actorId}/runs")
                .queryParam("webhooks", encodeWebhook())
                .build(actorId))
            .bodyValue(input)
            .retrieve()
            .toBodilessEntity()
            .block();

        log.info("Apify Actor 실행 요청 완료");
    }

    /** Dataset에서 결과 가져오기 */
    public List<Map<String, Object>> fetchItems(String datasetId) {
        return webClient.get()
            .uri("/datasets/{id}/items?clean=true", datasetId)
            .retrieve()
            .bodyToFlux(new ParameterizedTypeReference<Map<String, Object>>() {})
            .collectList()
            .block();
    }

    /** Apify는 webhooks 파라미터를 base64 인코딩된 JSON으로 받는다 */
    private String encodeWebhook() {
        String json = """
            [{
              "eventTypes": ["ACTOR.RUN.SUCCEEDED", "ACTOR.RUN.FAILED"],
              "requestUrl": "%s/api/apify/webhook"
            }]
            """.formatted(publicUrl);
        return Base64.getEncoder().encodeToString(json.getBytes(StandardCharsets.UTF_8));
    }
}
```

### 3) Webhook 수신 컨트롤러 — 핵심

```java
@RestController
@RequestMapping("/api/apify")
@RequiredArgsConstructor
@Slf4j
public class ApifyWebhookController {

    private final ApifyService apifyService;
    private final ScrapedDataService scrapedDataService;

    @Value("${apify.webhook-secret}")
    private String expectedSecret;

    @PostMapping("/webhook")
    public ResponseEntity<Void> handleWebhook(
        @RequestHeader(value = "X-Apify-Webhook-Secret", required = false) String secret,
        @RequestBody Map<String, Object> payload
    ) {
        // ① 시크릿 검증 — 외부 노출 엔드포인트라 필수
        if (!expectedSecret.equals(secret)) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }

        Map<String, Object> resource = (Map<String, Object>) payload.get("resource");
        String status = (String) resource.get("status");
        String datasetId = (String) resource.get("defaultDatasetId");

        log.info("Apify webhook 수신: status={}", status);

        // ② 빠르게 200 응답 후 비동기 처리 (Apify 재전송 방지)
        if ("SUCCEEDED".equals(status)) {
            scrapedDataService.saveAsync(datasetId);
        }

        return ResponseEntity.ok().build();
    }
}
```

### 4) 비동기 저장 서비스

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ScrapedDataService {

    private final ApifyService apifyService;
    private final ScrapedItemRepository repository;

    @Async
    @Transactional
    public void saveAsync(String datasetId) {
        List<Map<String, Object>> items = apifyService.fetchItems(datasetId);

        List<ScrapedItem> entities = items.stream()
            .map(item -> new ScrapedItem(
                (String) item.get("url"),
                (String) item.get("title")
            ))
            .toList();

        repository.saveAll(entities);
        log.info("저장 완료: {} 건", entities.size());
    }
}
```

> `@Async`를 쓰려면 메인 클래스에 `@EnableAsync`를 붙여야 한다.
> 

### 코드 분석

- **Spring 코드 어디에도 Crawlee, CheerioCrawler, PlaywrightCrawler가 등장하지 않는다.** Apify는 "결과를 돌려주는 외부 API"로만 다룬다. 카카오 로그인 API를 호출하는 것과 동일한 구조다.
- **Webhook 패턴이 동기 호출보다 우월한 이유**: Apify Actor 실행은 길게는 수십 분도 걸린다. Spring 스레드를 그동안 점유할 수는 없다. 비동기 실행 + 콜백으로 백엔드 자원을 효율적으로 쓴다.
- **Webhook 시크릿 검증은 필수다.** `/api/apify/webhook`은 인터넷에 공개된 엔드포인트이므로, 시크릿이 없으면 누구나 가짜 데이터를 주입할 수 있다.
- **`@Async`로 분리한 이유**: webhook 핸들러는 빠르게 200을 응답해야 한다. Apify는 일정 시간 내에 응답을 받지 못하면 webhook을 재전송하므로, 무거운 작업을 동기로 처리하면 중복 호출이 발생한다. 응답은 즉시 보내고 실제 처리는 별도 스레드에서 한다.
- **로컬 개발 시 webhook 수신**: Apify는 인터넷에서 접근 가능한 URL로만 webhook을 보낼 수 있다. 로컬 개발 중에는 `ngrok http 8080`으로 로컬 서버를 외부에 노출시킨 뒤 그 URL을 `app.public-url`에 넣는다.

## 💡 회고 및 질의응답

**회고**

- "크롤링"과 "스크래핑"을 같은 의미로 쓰던 습관이 있었는데, 이번에 명확히 구분하게 되었다. 코드 안에서도 두 행위가 분리되어 있다는 것을 보고 이해가 더 깊어졌다.
- Apify의 가장 큰 가치는 "크롤링 코드 자체"가 아니라 그 주변의 운영 인프라(프록시, 스토리지, 스케줄링, 모니터링)를 한 번에 해결해 준다는 점이라는 것을 알게 되었다.
- 처음에는 "강력한 도구일수록 좋다"고 생각해 무조건 Playwright + 레지덴셜 프록시를 떠올렸지만, 비용·복잡도 우선순위를 보고 생각이 바뀌었다. **가장 싼 방법부터 시도하고 막힐 때만 올라가는 것**이 핵심 원칙이다.
- 내부 API 역추적이 안티봇 우회의 가장 강력한 전략이라는 점이 인상적이었다. 브라우저를 띄우는 것보다 개발자도구 Network 탭을 분석하는 게 더 효과적일 수 있다는 발상의 전환이 필요하다.

**질의응답**

- **Q. CheerioCrawler와 PlaywrightCrawler의 분기는 어떻게 판단하는가?**
A. 브라우저에서 "페이지 소스 보기"를 했을 때 원하는 데이터가 HTML에 그대로 들어 있으면 Cheerio로 충분하다. 비어 있거나 `<div id="root"></div>` 같은 빈 컨테이너만 보인다면 JS 렌더링이 필요하므로 Playwright가 필요하다.
- **Q. 레지덴셜 프록시는 항상 비싸다고 들었는데, 비용 최적화 방법이 있나?**
A. 레지덴셜 프록시는 트래픽(GB) 단위로 과금되므로, 이미지·CSS·폰트 같은 불필요한 리소스 로딩을 차단하면 비용을 크게 줄일 수 있다. Playwright의 `route.abort()`를 활용한다. 또한 차단되지 않는 페이지는 데이터센터 프록시로, 차단되는 페이지만 레지덴셜로 보내는 하이브리드 전략도 가능하다.
- **Q. 크롤링 결과 데이터 구조가 사이트 변경으로 깨지면 어떻게 대응해야 하나?**
A. 출력에 대한 스키마 검증(Zod, Pydantic 등)을 넣어두고, 필수 필드 누락 시 알림이 가도록 구성한다. 셀렉터 기반 크롤링은 본질적으로 깨지기 쉬우므로, 가능하면 내부 API 응답이나 `data-*` 속성, `aria-label` 같은 안정적인 식별자를 활용하는 것이 좋다.
- **Q. Apify 없이 Crawlee만 써도 되는가?**
A. 가능하다. Crawlee는 독립 오픈소스 라이브러리로, 로컬이나 자체 서버에서 그대로 돌릴 수 있다. Apify는 운영 인프라(스케줄링, 프록시, 스토리지, 모니터링)를 제공해 줄 뿐이다. 소규모 프로젝트라면 Crawlee만으로 시작하고, 운영 부담이 커지면 Apify로 옮기는 전략이 현실적이다.
- **Q. SNS 크롤링이 합법인지 불법인지 한 번에 알 수 있는 기준이 있나?**
A. 단순한 기준은 없지만, 다음 조건을 모두 만족하면 위험이 낮다. ① 로그인 없이 접근 가능한 공개 데이터, ② 개인정보 미포함, ③ ToS에서 명시적으로 금지하지 않음, ④ 합리적인 요청 빈도, ⑤ 저작권 침해 없음. 하나라도 어긋나면 법무 검토가 필요하다.
