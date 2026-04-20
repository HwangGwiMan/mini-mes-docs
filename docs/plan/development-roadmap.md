# 개발 로드맵

---

## 개발 이력

### 2026-04-20
**Phase 3 — 작업지시 라우팅 전개(WorkOrderRouting) 추가 예정**

| 구분 | 백엔드 | 프론트엔드 |
|------|--------|------------|
| 기준 정보 (공통코드·직원·품목·거래처·품목단가·공정) | ✅ 완성 | ✅ 완성 |
| 거래 업무 (견적→수주→출하→매출) | ✅ 완성 | ✅ 완성 |
| Phase 1 — BOM | ✅ 완성 | ✅ 완성 |
| Phase 1 — 라우팅 | ✅ 완성 | ✅ 완성 |
| Phase 1 — 창고 | ✅ 완성 | ✅ 완성 |
| Phase 2 — 구매요청 *(로드맵 외 추가)* | ✅ 완성 | ✅ 완성 |
| Phase 2 — 구매발주 | ✅ 완성 | ✅ 완성 |
| Phase 2 — 자재입고 | ✅ 완성 | ✅ 완성 |
| Phase 2 — 수주이행현황 *(로드맵 외 추가, 조회 전용)* | 🟡 부분 | ✅ 완성 |
| Phase 2 — 재고 원장 (inventory) | ✅ 완성 | ✅ 완성 |
| Phase 3 — 작업지시 | 🟡 진행중 | ✅ 완성 |
| Phase 3 — 자재 출고 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |
| Phase 3 — 생산실적 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |
| Phase 4 — 품질검사 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |

**작업 내용 (작업지시 라우팅 전개)**
- `workorder/domain`: `WorkOrderRouting` 엔티티 추가, `WorkOrder`에 `routings` 컬렉션 추가
- `routing/application`: `RoutingService.findStepsByBomId(bomId)` 조회 메서드 추가 (크로스 도메인 접근 규칙 준수)
- `workorder/application`: `WorkOrderService` 생성/수정 시 라우팅 전개 로직 추가
  - `bomId`로 라우팅 조회 → 없으면 스킵 (라우팅 없는 품목 허용)
  - `RoutingStep` → `WorkOrderRouting` 스냅샷 생성 (processId, stepOrder, standardTime 복사)
- DB 마이그레이션: `work_order_routing` 테이블 추가

**비고**
- 라우팅이 없는 품목도 작업지시 생성 가능 (자재 전개만 수행)
- 다음 작업 우선순위: Phase 3 — 자재 출고(MaterialIssue) 백엔드

---

### 2026-04-14
**Phase 3 — 작업지시(WorkOrder) 백엔드 + 프론트엔드 구현 완료**

| 구분 | 백엔드 | 프론트엔드 |
|------|--------|------------|
| 기준 정보 (공통코드·직원·품목·거래처·품목단가·공정) | ✅ 완성 | ✅ 완성 |
| 거래 업무 (견적→수주→출하→매출) | ✅ 완성 | ✅ 완성 |
| Phase 1 — BOM | ✅ 완성 | ✅ 완성 |
| Phase 1 — 라우팅 | ✅ 완성 | ✅ 완성 |
| Phase 1 — 창고 | ✅ 완성 | ✅ 완성 |
| Phase 2 — 구매요청 *(로드맵 외 추가)* | ✅ 완성 | ✅ 완성 |
| Phase 2 — 구매발주 | ✅ 완성 | ✅ 완성 |
| Phase 2 — 자재입고 | ✅ 완성 | ✅ 완성 |
| Phase 2 — 수주이행현황 *(로드맵 외 추가, 조회 전용)* | 🟡 부분 | ✅ 완성 |
| Phase 2 — 재고 원장 (inventory) | ✅ 완성 | ✅ 완성 |
| Phase 3 — 작업지시 | ✅ 완성 | ✅ 완성 |
| Phase 3 — 자재 출고 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |
| Phase 3 — 생산실적 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |
| Phase 4 — 품질검사 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |

**구현 내용 (작업지시)**
- `workorder/domain`: `WorkOrder`, `WorkOrderMaterial` 엔티티 + `WorkOrderStatus` enum(DRAFT/CONFIRMED/CANCELLED) + JPA Converter + `WorkOrderRepository`
- `workorder/application`: `WorkOrderService` — BOM 전개 스냅샷 생성, confirm(자재 선점), cancel(자재 해제)
- `workorder/internal`: `WorkOrderQueryRepository` — jOOQ raw DSL 목록/상세 조회
- `workorder/api`: `WorkOrderController` (7개 엔드포인트) + DTO 3종
- `inventory/application`: `InventoryService.reserveMaterial` / `unreserveMaterial` 스텁 완성
  - MATERIAL_RESERVE: Inventory.reserve(qty) + InventoryTx 기록
  - MATERIAL_UNRESERVE: Inventory.unreserve(qty) + InventoryTx 기록
- 프론트엔드: `workOrder.ts` API, `WorkOrderFormModal.vue`, `WorkOrderView.vue`
- 라우터: `/work-order` 추가, 사이드바: "생산 관리 > 작업지시" 메뉴 추가

**API 목록 (작업지시)**
| 메서드 | 경로 | 설명 |
|--------|------|------|
| GET | `/api/work-orders` | 목록 조회 (workOrderNumber, itemName, statusCode 필터) |
| GET | `/api/work-orders/{id}` | 상세 조회 (materials 포함) |
| POST | `/api/work-orders` | 생성 (DRAFT, BOM 전개 → WorkOrderMaterial 생성) |
| PUT | `/api/work-orders/{id}` | 수정 (DRAFT만 가능) |
| DELETE | `/api/work-orders/{id}` | 삭제 (DRAFT만 가능) |
| PATCH | `/api/work-orders/{id}/confirm` | 확정 (자재 reserveMaterial 호출) |
| PATCH | `/api/work-orders/{id}/cancel` | 취소 (CONFIRMED이면 unreserveMaterial 호출) |

**비고**
- 다음 작업 우선순위: Phase 3 — 자재 출고(MaterialIssue) 백엔드

---

### 2026-04-09
**재고 원장(Inventory) 프론트엔드 구현 완료**

| 구분 | 백엔드 | 프론트엔드 |
|------|--------|------------|
| 기준 정보 (공통코드·직원·품목·거래처·품목단가·공정) | ✅ 완성 | ✅ 완성 |
| 거래 업무 (견적→수주→출하→매출) | ✅ 완성 | ✅ 완성 |
| Phase 1 — BOM | ✅ 완성 | ✅ 완성 |
| Phase 1 — 라우팅 | ✅ 완성 | ✅ 완성 |
| Phase 1 — 창고 | ✅ 완성 | ✅ 완성 |
| Phase 2 — 구매요청 *(로드맵 외 추가)* | ✅ 완성 | ✅ 완성 |
| Phase 2 — 구매발주 | ✅ 완성 | ✅ 완성 |
| Phase 2 — 자재입고 | ✅ 완성 | ✅ 완성 |
| Phase 2 — 수주이행현황 *(로드맵 외 추가, 조회 전용)* | 🟡 부분 | ✅ 완성 |
| Phase 2 — 재고 원장 (inventory) | ✅ 완성 | ✅ 완성 |
| Phase 3 — 작업지시 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |
| Phase 3 — 자재 출고 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |
| Phase 3 — 생산실적 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |
| Phase 4 — 품질검사 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |

**구현 내용 (재고 원장 프론트엔드)**
- 라우트 `/inventory` 및 사이드바 메뉴 "재고 관리 > 재고 원장" 추가
- `InventoryView.vue`: 단일 뷰에 세 탭 구성 — 탭 간 창고·품목 필터 상태 공유
  - **현재고 탭**: 창고×품목 집계, 가용수량 색상 표시(양수: 녹색 / 0: 빨강), 이동·조정 버튼
  - **LOT별 탭**: LOT 단위 현재고, 유효기한 만료 시 빨강 / 30일 내 주황(D-N) 표시
  - **수불이력 탭**: 날짜 범위 필터, 수불유형 한글 라벨, 수량 증감 색상 표시, 날짜 미지정 시 안내 배너
- `InventoryTransferModal.vue`: 창고 간 이동 폼 (출발/도착 창고, 수량, 선택적 LOT 지정), 클라이언트 검증 포함
- `InventoryAdjustModal.vue`: 재고 조정 폼 (ADJUST_IN / ADJUST_OUT 라디오 선택), ADJUST_OUT은 ConfirmDialog 2차 확인
- `src/api/inventory.ts`: DTO 인터페이스 + `TX_TYPE_LABELS` 한글 라벨 맵 + API 메서드 5종

**비고**
- 다음 작업 우선순위: Phase 3 — 작업지시 백엔드

---

### 2026-04-08
**재고 원장(Inventory) 백엔드 구현 완료**

| 구분 | 백엔드 | 프론트엔드 |
|------|--------|------------|
| 기준 정보 (공통코드·직원·품목·거래처·품목단가·공정) | ✅ 완성 | ✅ 완성 |
| 거래 업무 (견적→수주→출하→매출) | ✅ 완성 | ✅ 완성 |
| Phase 1 — BOM | ✅ 완성 | ✅ 완성 |
| Phase 1 — 라우팅 | ✅ 완성 | ✅ 완성 |
| Phase 1 — 창고 | ✅ 완성 | ✅ 완성 |
| Phase 2 — 구매요청 *(로드맵 외 추가)* | ✅ 완성 | ✅ 완성 |
| Phase 2 — 구매발주 | ✅ 완성 | ✅ 완성 |
| Phase 2 — 자재입고 | ✅ 완성 | ✅ 완성 |
| Phase 2 — 수주이행현황 *(로드맵 외 추가, 조회 전용)* | 🟡 부분 | ✅ 완성 |
| Phase 2 — 재고 원장 (inventory) | ✅ 완성 | ✅ 완성 (2026-04-09) |
| Phase 3 — 작업지시 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |
| Phase 3 — 자재 출고 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |
| Phase 3 — 생산실적 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |
| Phase 4 — 품질검사 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |

**구현 내용 (재고 원장 백엔드)**
- `inventory/domain`: `Inventory`, `InventoryLot`, `InventoryTx` 엔티티 + Repository 3종 + `InventoryTxType` 열거형
  - `Inventory`·`InventoryLot`: BaseEntity 상속, 낙관적 락(@Version), 수량 조작 메서드(receive/issue/reserve/unreserve)
  - `InventoryTx`: 불변 원장 — BaseEntity 미상속, `@EntityListeners(AuditingEntityListener)` 직접 적용
- `inventory/application`: `InventoryService`(receiveStock/transfer/adjust + Phase D용 stub 5개) + `InventoryEventHandler`(`StockReceivedEvent` 수신 → receiveStock 위임)
- `inventory/internal`: `InventoryQueryRepository` — jOOQ raw DSL로 현재고·LOT·수불이력 조회
- `inventory/api`: `InventoryController` (5개 엔드포인트) + DTO 5종 (InventoryResponse, InventoryLotResponse, InventoryTxResponse, TransferRequest, AdjustRequest)
- `goodsreceipt`: 입고 확정 시 `StockReceivedEvent` 발행, DTO 패키지 구조 정리 (`api/` → `api/dto/`)
- `common/exception/GlobalExceptionHandler`: `ObjectOptimisticLockingFailureException` → HTTP 409 핸들러 추가

**API 목록 (재고 원장)**
| 메서드 | 경로 | 설명 |
|--------|------|------|
| GET | `/api/inventory` | 현재고 목록 (창고×품목 집계) |
| GET | `/api/inventory/lots` | LOT별 현재고 목록 |
| GET | `/api/inventory/transactions` | 수불 이력 (기간·창고·품목 필터) |
| POST | `/api/inventory/transfer` | 창고 간 재고 이동 |
| POST | `/api/inventory/adjust` | 재고 조정 (ADJUST_IN / ADJUST_OUT) |

**비고**
- Phase 3 연동용 stub(`reserveMaterial`, `unreserveMaterial`, `issueMaterial`, `receiveProduction`, `issueSales`)은 InventoryService에 시그니처만 정의, 작업지시 도메인 구현 시 완성 예정
- 다음 작업 우선순위: Phase 3 — 작업지시 백엔드 (재고 원장 프론트엔드는 2026-04-09 완료)

---

### 2026-04-01
**진척 상황 전체 점검 (백엔드 구현 깊이 포함)**

| 구분 | 백엔드 | 프론트엔드 |
|------|--------|------------|
| 기준 정보 (공통코드·직원·품목·거래처·품목단가·공정) | ✅ 완성 | ✅ 완성 |
| 거래 업무 (견적→수주→출하→매출) | ✅ 완성 | ✅ 완성 |
| Phase 1 — BOM | ✅ 완성 | ✅ 완성 |
| Phase 1 — 라우팅 | ✅ 완성 | ✅ 완성 |
| Phase 1 — 창고 | ✅ 완성 | ✅ 완성 |
| Phase 2 — 구매요청 *(로드맵 외 추가)* | ✅ 완성 | ✅ 완성 |
| Phase 2 — 구매발주 | ✅ 완성 | ✅ 완성 |
| Phase 2 — 자재입고 | ✅ 완성 | ✅ 완성 |
| Phase 2 — 수주이행현황 *(로드맵 외 추가, 조회 전용)* | 🟡 부분 | ✅ 완성 |
| Phase 2 — 재고 원장 (inventory) | ✅ 완성 (2026-04-08) | ❌ 미구현 |
| Phase 3 — 작업지시 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |
| Phase 3 — 자재 출고 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |
| Phase 3 — 생산실적 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |
| Phase 4 — 품질검사 | ❌ 미구현 (패키지·설계 주석만) | ❌ 미구현 |

**비고**
- Phase 3~4 전체는 패키지 뼈대 및 설계 주석만 존재하며 실제 로직 없음
- 다음 작업 우선순위: 재고 원장(inventory) 프론트엔드 또는 Phase 3 — 작업지시 백엔드

---

## 현재 개발된 업무 영역

### 기준 정보
- 공통코드, 직원, 품목, 거래처, 품목단가, 공정

### 거래 업무
- 견적 → 수주 → 출하 → 매출

---

## 추천 개발 순서

### Phase 1 — 기준 정보 보완 (선행 필수)
| 영역 | 이유 |
|------|------|
| **BOM (자재명세서)** | 품목 생산에 필요한 자재와 수량 정의. 생산/자재 소요 계산의 근거 |
| **라우팅 (공정순서)** | 품목별 공정 순서 정의 (기존 공정 마스터 활용) |
| **창고** | 재고 관리의 기준 단위. 삭제 시 재고 존재 여부 검증 필요 |

### Phase 2 — 재고 관리
| 영역 | 주요 기능 |
|------|-----------|
| **재고 기준 설정** | 품목×창고×LOT 단위 현재고 관리, 수불 이력 원장 |
| **구매발주** | BOM 기반 자재 소요 계산 → 거래처에 발주 |
| **자재입고** | 발주 대비 입고 처리, LOT 지정 입고, 재고 반영 |
| **재고 현황** | 현재고 조회(창고×품목×LOT), 수불 이력, 창고 간 이동, 재고 조정 |

> 재고 수불 설계 결정사항: [ADR-004](../adr/004-inventory-transaction-design.md) 참조

### Phase 3 — 생산 관리
| 영역 | 주요 기능 |
|------|-----------|
| **작업지시** | 수주 품목 기반 BOM 전개, N건 작업지시 생성, 투입 자재 선점 |
| **자재 출고** | 작업지시별 투입 자재 LOT 지정, 출고 확정 (생산 시작 필수 조건) |
| **생산실적** | 작업지시 대비 실제 생산량/불량수량 입력, 완제품 재고 반영 |
| **출고 등록** | 수주 연결, 완제품 LOT 지정, 출고 확정 |

### Phase 4 — 품질 (선택)
| 영역 | 주요 기능 |
|------|-----------|
| **품질검사** | 입고 검사, 공정 검사, 완성품 검사 |

---

## 완성 후 전체 업무 흐름

```
[수주]
  ↓
[작업지시 등록] ← BOM 전개 (N건, 다단계)
  ↓  투입 자재 선점 (qty_reserved++)
[자재 출고 등록] → LOT 지정 → [출고 확정] → 자재 재고 차감
  ↓  (미확정 시 생산 시작 BLOCK)
[생산 시작] → [생산실적] → 완제품/반제품 재고 입고
  ↓
[출고 등록] → LOT 지정 → [출고 확정] → 완제품 재고 차감
  ↓
[매출]
```

### 재고 이벤트 요약
| 단계 | 재고 변화 |
|------|----------|
| 자재입고 확정 | 원자재 `qty_on_hand++` |
| 작업지시 등록 | 투입 자재 `qty_reserved++` |
| 자재 출고 확정 | 투입 자재 `qty_on_hand--`, `qty_reserved--` |
| 생산 완료 | 완제품/반제품 `qty_on_hand++` |
| 출고 확정 | 완제품 `qty_on_hand--` |
| 창고 간 이동 | 출발 `qty_on_hand--`, 도착 `qty_on_hand++` |
