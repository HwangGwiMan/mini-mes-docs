# ADR-008: 테스트 레이어 아키텍처

## 상태
승인됨

## 날짜
2026-04-17

## 맥락

프로젝트가 Phase 3(작업지시) 완료 시점에 도달했고, 이후 자재출고(MaterialIssue), 생산실적(ProductionResult), 품질검사(QualityInspection) 구현이 예정되어 있다. 지금까지는 Spring Modulith 모듈 격리 테스트(12개)만 존재하며 서비스 레이어와 API 레이어, 프론트엔드에는 테스트 코드가 없다.

이 프로젝트의 핵심 위험 요소는 세 가지다:

1. **재고 스냅샷의 공유 상태**: `Inventory.qtyOnHand`, `qtyReserved`는 입고확정·작업지시확정·자재출고·창고이전 4개 흐름이 동시에 변경하며, 낙관적 잠금이 유일한 동시성 보호 수단이다.
2. **이벤트 체인의 비가시성**: `SalesOrderCreatedEvent → ShipmentService.on()` 경로는 단위 테스트로는 검증 불가능하며 실제 Spring Context가 필요하다.
3. **상태 기계 회귀**: `QuoteStatus`, `WorkOrderStatus` 등 도메인 상태 전이 규칙은 Enum 리팩토링 한 번에 조용히 깨질 수 있다.

이를 근거로 5개 레이어의 테스트 아키텍처를 결정한다.

---

## 검토한 선택지

- **선택지 A**: 단위 테스트(Mock) 위주 — 빠르지만 재고 로직과 이벤트 체인을 검증할 수 없음
- **선택지 B**: 통합 테스트만 — DB 의존도 높고 실행 느림, 빠른 피드백 불가
- **선택지 C**: 5개 레이어 계층형 — 각 레이어가 명확한 목적을 가지며 빠른 피드백과 신뢰성을 동시에 제공

---

## 결정

**선택지 C — 5개 레이어 계층형 테스트 구조**를 채택한다.

### 레이어 정의

| 레이어 | 도구 | 목적 | DB | 속도 |
|---|---|---|---|---|
| L1: 모듈 격리 | Spring Modulith `@ApplicationModuleTest` | 모듈 경계 위반 감지 | 없음 | 매우 빠름 |
| L2: 서비스 통합 | `@SpringBootTest` + Testcontainers | 유스케이스, 이벤트 체인, 낙관적 잠금 | 실제 PostgreSQL | 느림 |
| L3: 컨트롤러 슬라이스 | `@WebMvcTest` + MockitoBean | HTTP 상태코드, Bean Validation, 인증 | 없음 (Mock) | 빠름 |
| L4: 프론트 유닛 | Vitest + @vue/test-utils | Composable 로직, API 함수 | 없음 (MSW) | 매우 빠름 |
| L5: E2E | Playwright | 업무 황금 경로 end-to-end | 실제 DB (시드) | 가장 느림 |

---

## 레이어별 규격

### L1: 모듈 격리 테스트

**위치:** `src/test/java/.../{module}/`  
**파일명:** `{Domain}ModuleTest.java`

```java
@ApplicationModuleTest(mode = BootstrapMode.STANDALONE)
class WorkOrderModuleTest {
    @Test void contextLoads() {}
}
```

**규칙:**
- 모든 도메인 모듈은 반드시 `contextLoads()` 테스트를 가진다
- `allowedDependencies`에 없는 모듈 참조 시 컨텍스트 로드 실패로 경계 위반을 즉시 감지

---

### L2: 서비스 통합 테스트

**위치:** `src/test/java/.../{module}/application/`  
**파일명:** `{Domain}ServiceIntegrationTest.java`  
**공통 기반 클래스:** `support/ServiceIntegrationTest.java`

```java
// support/ServiceIntegrationTest.java
@SpringBootTest(webEnvironment = NONE)
@Testcontainers
@ActiveProfiles("test")
@Transactional   // 테스트별 자동 롤백
abstract class ServiceIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureDataSource(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", postgres::getJdbcUrl);
        r.add("spring.datasource.username", postgres::getUsername);
        r.add("spring.datasource.password", postgres::getPassword);
    }
}
```

**픽스처 전략:**
- Fixture Monkey 미사용. 도메인 엔티티는 `AccessLevel.PROTECTED` 생성자 + 팩토리 메서드 패턴이므로 반사 생성이 불가능하다.
- 대신 모듈별 `TestFixtures` 헬퍼 클래스를 작성해 팩토리 메서드를 직접 호출한다.

```java
// quote/application/QuoteTestFixtures.java
class QuoteTestFixtures {
    static Quote createDraftQuote(Long partnerId, Long employeeId, ...) {
        return Quote.create(partnerId, employeeId, ...);  // 실제 팩토리 메서드 호출
    }
}
```

**`@Transactional` 예외:**
이벤트 체인(`SalesOrderCreatedEvent → ShipmentService.on()`)은 `@ApplicationModuleListener`로 동일 트랜잭션 내 동기 실행되므로 기본 `@Transactional` 전략으로 검증 가능하다.

**필수 build.gradle 의존성:**
```groovy
testImplementation 'org.springframework.boot:spring-boot-testcontainers'
testImplementation 'org.testcontainers:junit-jupiter'
testImplementation 'org.testcontainers:postgresql'
```

**우선순위 구현 순서:**

| 순위 | 클래스 | 핵심 이유 |
|---|---|---|
| 1 | `InventoryServiceIntegrationTest` | 재고 스냅샷 공유 상태, 낙관적 잠금 |
| 2 | `WorkOrderServiceIntegrationTest` | confirm() 크로스 모듈 재고 예약 |
| 3 | `SalesOrderEventChainIntegrationTest` | 이벤트 체인 비가시성 |
| 4 | `QuoteServiceIntegrationTest` | 상태 전이, 문서번호 채번 |
| 5 | `GoodsReceiptServiceIntegrationTest` | StockReceivedEvent → 재고 갱신 |
| 6 | `MaterialIssueServiceIntegrationTest` | Phase 3 구현 시 `@Disabled` 해제 |

---

### L3: 컨트롤러 슬라이스 테스트

**위치:** `src/test/java/.../{module}/api/`  
**파일명:** `{Domain}ControllerTest.java`

```java
@WebMvcTest(QuoteController.class)
@Import(SecurityConfig.class)
class QuoteControllerTest {

    @Autowired MockMvc mockMvc;
    @MockitoBean QuoteService quoteService;

    @Test
    @WithMockUser
    void create_validRequest_returns201() throws Exception {
        // given
        given(quoteService.create(any())).willReturn(stubResponse());
        // when & then
        mockMvc.perform(post("/api/quotes").contentType(APPLICATION_JSON).content("..."))
               .andExpect(status().isCreated());
    }

    @Test
    void create_withoutAuth_returns403() throws Exception {
        mockMvc.perform(post("/api/quotes").contentType(APPLICATION_JSON).content("..."))
               .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser
    void submit_businessRuleViolation_returns409() throws Exception {
        given(quoteService.submit(anyLong(), any()))
            .willThrow(new BusinessRuleViolationException("이미 제출된 견적입니다."));
        mockMvc.perform(patch("/api/quotes/1/submit"))
               .andExpect(status().isConflict());
    }
}
```

**규칙:**
- `GlobalExceptionHandler`는 `@RestControllerAdvice`이므로 `@WebMvcTest` 컨텍스트에 자동 포함됨. 별도 설정 불필요.
- 마스터 데이터 컨트롤러(Partner, Item, Employee, Warehouse, CommonCode, Process)는 작성하지 않는다. 순수 CRUD로 비즈니스 로직이 없다.
- HTTP 상태코드 매핑은 [테스트 전략 계획](../plan/testing-strategy.md)의 기준표를 따른다.

---

### L4: 프론트엔드 유닛 테스트

**위치:** `src/composables/__tests__/`, `src/api/__tests__/`, `src/stores/__tests__/`  
**파일명:** `{target}.spec.ts`

**설정 파일:** `vite.config.ts`

```ts
export default defineConfig({
  // ...기존 설정...
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
  },
})
```

**API 모킹:** MSW(Mock Service Worker)로 axios 요청을 인터셉트한다. axios mock adapter 대신 MSW를 사용하는 이유: 실제 네트워크 레이어를 통과하므로 인터셉터(`401 → logout`) 동작까지 검증 가능하다.

```ts
// src/test/setup.ts
import { setupServer } from 'msw/node'
export const server = setupServer()
beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

**최우선 대상:** `useCrudPage.ts` — 21개 CRUD 화면이 사용하는 공통 composable. 한 번의 테스트 스위트로 전체 UI 상호작용 패턴을 커버한다.

---

### L5: E2E 테스트

**위치:** `e2e/` (프론트엔드 루트)  
**파일명:** `{flow}.spec.ts`

**설정:** `playwright.config.ts`

```ts
export default defineConfig({
  testDir: './e2e',
  use: { baseURL: 'http://localhost:5173' },
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:5173',
    reuseExistingServer: !process.env.CI,
  },
})
```

**테스트 데이터 전략:**  
`test` Spring 프로파일 전용 `E2ETestDataController`를 구현한다. `DataInitializer`는 시작 시 1회만 실행되어 E2E 반복 실행에 부적합하다.

```
POST /api/test/seed    → 고정 ID 픽스처 삽입, 삽입된 ID JSON 반환
POST /api/test/cleanup → 도메인 테이블 초기화
```

**황금 경로 3개만 구현한다:**
1. Quote → SalesOrder → Shipment 자동 생성
2. 작업지시 확정 → 재고 예약 → 취소 → 예약 해제
3. 입고 확정 → 재고 반영

E2E는 목록/검색 화면, 마스터 데이터 CRUD는 대상에서 제외한다. 비즈니스 로직이 없는 화면의 E2E는 유지비 대비 가치가 낮다.

---

## 결과

### 긍정적
- 재고 낙관적 잠금 동작을 L2에서 실제 DB로 검증 가능
- 이벤트 체인 회귀를 L2에서 자동으로 감지
- L3은 DB 없이 빠르게 HTTP 계약을 검증
- L4는 useCrudPage 한 번으로 21개 화면 커버
- L5는 황금 경로만 선별하여 유지비 최소화

### 부정적 / 트레이드오프
- L2는 Testcontainers로 Docker가 필요하며 실행이 느림 (CI에서 허용 가능한 수준)
- L5 E2E는 백엔드 + 프론트엔드 동시 실행 환경이 필요
- Fixture Monkey 미사용으로 픽스처 코드를 직접 작성해야 함 (가독성은 더 좋음)

---

## 관련 문서

- [테스트 전략 계획](../plan/testing-strategy.md)
- [Layer 2 서비스 통합 테스트 명세](../../backend/docs/plan/layer2-service-integration-test.md)
- [Layer 3 컨트롤러 슬라이스 테스트 명세](../../backend/docs/plan/layer3-controller-slice-test.md)
- [ADR-005: 도메인 상태 코드를 Enum으로 관리](./005-domain-status-enum.md)
- [ADR-004: 재고 트랜잭션 설계](./004-inventory-transaction-design.md)
