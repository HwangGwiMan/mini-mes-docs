# 테스트 전략 계획

## Context

Phase 3(작업지시) 완료 시점 기준으로, 프로젝트에는 Spring Modulith 모듈 격리 테스트 12개만 존재한다. 서비스 통합 테스트와 컨트롤러 슬라이스 테스트는 계획 문서만 있고 미구현 상태이며, 프론트엔드는 테스트 코드가 전무하다.

이 문서는 전체 프로젝트에 걸쳐 5개 레이어의 테스트를 어떤 순서로, 어떤 기준으로 작성할지를 정의한다.

---

## 테스트 레이어 전체 구조

```
Layer 5  E2E (Playwright)               — 핵심 업무 흐름 3개
Layer 4  프론트엔드 유닛 (Vitest)        — composable / API 함수
Layer 3  컨트롤러 슬라이스 (@WebMvcTest) — HTTP 상태코드, Bean Validation, 인증
Layer 2  서비스 통합 (Testcontainers)    — 실제 DB, 이벤트 체인, 낙관적 잠금
Layer 1  모듈 격리 (Spring Modulith)     — 경계 검증 (부분 완료)
```

---

## Layer 1 — 모듈 격리 테스트 (갭 보완)

### 현황

`@ApplicationModuleTest(STANDALONE)` 완료된 모듈: `auth, item, itemprice, partner, process, employee, commoncode, quote, salesorder, shipment` (10개)

### 추가 필요 모듈 (8개)

| 모듈 | 이유 |
|---|---|
| `workorder` | `bom`, `inventory`, `item`, `warehouse` 의존 |
| `inventory` | 3개 이상 모듈의 이벤트 수신 |
| `goodsreceipt` | `purchaseorder`, `inventory` 이벤트 연동 |
| `bom` | `workorder`가 의존 |
| `routing` | `production`이 의존 |
| `purchaserequest` | `purchaseorder`가 의존 |
| `revenue` | `salesorder`, `shipment` 교차 참조 |
| `materialissue` | Phase 3 신규 구현 예정 |

### 구현 방법

각 모듈에 `XxxModuleTest.java` 클래스를 추가한다. DB 없이 수 초 내 완료.

```java
@ApplicationModuleTest(mode = BootstrapMode.STANDALONE)
class WorkOrderModuleTest {
    @Test void contextLoads() {}
}
```

**예상 테스트 수: 8개**

---

## Layer 2 — 서비스 통합 테스트

상세 명세는 [layer2-service-integration-test.md](../../backend/docs/plan/layer2-service-integration-test.md) 참조.

### 우선순위

재고 스냅샷(`qty_on_hand`, `qty_reserved`)은 4개 이상의 흐름이 공유하는 가장 위험한 상태이므로 최우선으로 다룬다.

| 우선순위 | 테스트 클래스 | 핵심 검증 | 예상 수 |
|---|---|---|---|
| P1 | `InventoryServiceIntegrationTest` | qty 증감, 낙관적 잠금 동시 충돌 | 12 |
| P2 | `WorkOrderServiceIntegrationTest` | confirm() → 자재예약, 취소 → 예약해제 | 8 |
| P3 | `SalesOrderEventChainIntegrationTest` | SalesOrderCreatedEvent → Shipment 자동생성 | 6 |
| P4 | `QuoteServiceIntegrationTest` | 상태 전이, 승인 이력, 문서번호 채번 | 9 |
| P5 | `GoodsReceiptServiceIntegrationTest` | confirm → StockReceivedEvent → 재고 갱신 | 5 |
| P6 | `MaterialIssueServiceIntegrationTest` | qty_on_hand--, qty_reserved-- (Phase 3 병행, @Disabled) | 4 |

**Layer 2 예상 테스트 수: ~44개**

---

## Layer 3 — 컨트롤러 슬라이스 테스트

상세 명세는 [layer3-controller-slice-test.md](../../backend/docs/plan/layer3-controller-slice-test.md) 참조.

### HTTP 상태코드 매핑 기준

| 코드 | 원인 |
|---|---|
| 201 | 생성 성공 |
| 204 | 수정/삭제/액션 성공 |
| 400 | `@Valid` Bean Validation 실패 |
| 401/403 | 인증/인가 실패 |
| 404 | `ResourceNotFoundException` |
| 409 | `BusinessRuleViolationException` |

### 우선순위

| 테스트 클래스 | 핵심 시나리오 | 예상 수 |
|---|---|---|
| `QuoteControllerTest` | 10개 엔드포인트, 상태코드 전체 검증 | 9 |
| `WorkOrderControllerTest` | confirm 409, cancel 204 | 5 |
| `SalesOrderControllerTest` | convertFromQuote 401/409 | 7 |
| `ShipmentControllerTest` | complete 상태 가드 | 5 |
| `InventoryControllerTest` | adjust 유효성 검증 | 5 |
| `AuthControllerTest` | JWT 발급, 잘못된 비밀번호 401 | 3 |

> 마스터 데이터 컨트롤러(Partner, Item, Employee 등)는 Controller 테스트를 생략한다. 순수 CRUD로 비즈니스 로직이 없으며, 통합 테스트에서 픽스처로 이미 실행된다.

**Layer 3 예상 테스트 수: ~34개**

---

## Layer 4 — 프론트엔드 유닛 테스트 (Vitest)

### 설치

```bash
npm install -D vitest @vue/test-utils jsdom @vitest/coverage-v8 msw
```

`vite.config.ts`에 추가:

```ts
test: {
  environment: 'jsdom',
  globals: true,
  setupFiles: ['./src/test/setup.ts'],
}
```

### 우선순위

| 파일 | 대상 | 예상 수 |
|---|---|---|
| `useCrudPage.spec.ts` | 21개 CRUD 화면이 사용하는 공통 composable | 9 |
| `quote.spec.ts` | submit/approve/reject HTTP 요청 형태 (MSW) | 5 |
| `api-error.spec.ts` | `extractErrorMessage` 엣지케이스 | 4 |
| `auth.spec.ts` | Pinia auth store token 관리 | 3 |
| `useColumnSettings.spec.ts` | localStorage 기반 컬럼 설정 persist/restore | 3 |

**Layer 4 예상 테스트 수: ~24개**

---

## Layer 5 — E2E (Playwright)

### 설치

```bash
npx playwright install chromium
npm install -D @playwright/test
```

### 테스트 데이터 전략

`test` 프로파일 전용 `E2ETestDataController` 구현:
- `POST /api/test/seed` — 고정 ID 픽스처 삽입, 삽입된 ID JSON 반환
- `POST /api/test/cleanup` — 도메인 테이블 초기화

`beforeEach`에서 호출. `DataInitializer`는 시작 시 1회만 실행되어 E2E 반복 실행에 부적합하다.

### 황금 경로 3개

#### GP1: Quote → SalesOrder → Shipment 자동 생성
**파일:** `e2e/quote-to-shipment.spec.ts`

```
1. admin 로그인
2. /quotes → 신규 견적 생성 (픽스처: 파트너, 품목, 담당자, 승인자)
3. 견적 제출 → 상태 칩 "제출됨" 확인
4. 승인자 계정으로 로그인 → 견적 승인
5. 수주 전환 → /orders에 SO 번호 확인
6. /shipments → 자동 생성된 출하 확인 (상태: "계획", 라인 수 일치)
```

#### GP2: 작업지시 확정 → 재고 예약 → 취소 → 예약 해제
**파일:** `e2e/workorder-confirm.spec.ts`

```
1. 시드: 원자재(qty=100), BOM(2라인), 창고, 재고
2. 작업지시 생성 (plannedQty=10)
3. 자재 목록 확인: qty = BOM라인qty × 10
4. "확정" → 재고 화면에서 qty_reserved 증가 확인
5. 작업지시 취소 → qty_reserved 원복 확인
```

#### GP3: 입고 확정 → 재고 반영
**파일:** `e2e/goods-receipt-confirm.spec.ts`

```
1. 시드: 승인된 PO, 품목, 창고 (재고 qty=0)
2. 입고 생성 (qty=50) → 입고 확정
3. PO 상태 "입고완료" 확인
4. 재고 화면 → qty_on_hand=50, TX 타입 PURCHASE_IN 확인
```

---

## 구현 순서 (스프린트)

| 스프린트 | 작업 | 기간 |
|---|---|---|
| S1 | build.gradle Testcontainers 추가, AbstractIntegrationTest 작성, InventoryServiceIntegrationTest, Layer 1 갭 8개 | ~3일 |
| S2 | WorkOrderServiceIntegrationTest, SalesOrderEventChainIntegrationTest, QuoteServiceIntegrationTest | ~3일 |
| S3 | Layer 3 컨트롤러 테스트 전체 (6개 클래스) | ~2일 |
| S4 | Vitest 설치, useCrudPage.spec.ts, 나머지 유닛 테스트 | ~2일 |
| S5 | Playwright 설치, E2ETestDataController, 황금 경로 3개 | ~2일 |
| Phase3 병행 | MaterialIssueServiceIntegrationTest @Disabled 선언 → 서비스 구현 시 활성화 | - |

---

## 예상 테스트 수 요약

| 레이어 | 예상 수 |
|---|---|
| Layer 1 (갭 보완) | 8 |
| Layer 2 (서비스 통합) | ~44 |
| Layer 3 (컨트롤러) | ~34 |
| Layer 4 (프론트 유닛) | ~24 |
| Layer 5 (E2E) | 3 파일 (황금 경로) |
| **총합** | **~110+ 테스트** |

---

## 의도적으로 제외하는 항목

- 마스터 데이터 컨트롤러 슬라이스 테스트 — 순수 CRUD, 비즈니스 로직 없음
- `QuoteStatus.from()` 단독 유닛 테스트 — 모든 통합 테스트에서 자연스럽게 실행됨
- `DocumentNumberGenerator` 단독 테스트 — DB 시퀀스 의존, 통합 테스트에서 실행됨
- E2E 목록/검색 화면 — 비즈니스 로직 없음
- Fixture Monkey — 도메인 엔티티의 `AccessLevel.PROTECTED` 생성자 + 팩토리 메서드 패턴과 맞지 않음. 모듈별 `TestFixtures` 헬퍼 클래스로 대체

---

## 관련 문서

- [ADR-008: 테스트 레이어 아키텍처](../adr/008-test-layer-architecture.md)
- [Layer 2 서비스 통합 테스트 명세](../../backend/docs/plan/layer2-service-integration-test.md)
- [Layer 3 컨트롤러 슬라이스 테스트 명세](../../backend/docs/plan/layer3-controller-slice-test.md)
