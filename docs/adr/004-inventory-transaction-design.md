# ADR-004: 재고 수불 설계

## 상태
승인됨

## 날짜
2026-03-30

## 맥락
재고 관리 업무 개발에 앞서, 현재고를 어떻게 유지하고 수불 이력을 어떻게 추적할지 설계 방향을 결정해야 했다.
주요 고려 사항:
- 창고 단위 재고 관리
- LOT 단위 추적
- 수주/작업지시와 연계된 재고 선점(예약)
- 음수 재고 방지
- 창고 간 이동
- BOM 다단계 구조에 따른 작업지시 N건 생성

## 검토한 선택지

- **선택지 A — 원장(Ledger) 방식**: 수불 이력만 적재, 현재고는 SUM 집계
- **선택지 B — 스냅샷(Snapshot) 방식**: `inventory` 테이블에 현재고 직접 유지
- **선택지 C — 혼합 방식**: 스냅샷 + 원장 동시 유지, 트랜잭션으로 원자적 처리

## 결정

**선택지 C (혼합 방식)** 을 채택한다.

현재고 빠른 조회를 위해 스냅샷 테이블을 유지하고, 완전한 감사 추적을 위해 수불 이력 원장을 함께 운영한다.
낙관적 락(`@Version`)으로 동시 업데이트 충돌을 방지한다.

### 테이블 구조

```
warehouse              창고 기준정보 (CRUD, 삭제 시 재고 존재 여부 검증)

inventory              품목 × 창고 집계 현재고
  warehouse_id, item_id
  qty_on_hand          실물 보유량
  qty_reserved         선점(예약)량 — 작업지시 등록 시 투입 자재에 대해 설정
  version              낙관적 락

inventory_lot          품목 × 창고 × LOT 현재고 (전 품목 LOT 관리)
  warehouse_id, item_id, lot_no
  qty_on_hand
  qty_reserved
  expiry_date
  version

inventory_tx           수불 이력 원장 (불변)
  warehouse_id, item_id, lot_no
  tx_type
  qty_delta            항상 양수, 방향은 tx_type으로 결정
  ref_type             출처 유형 (SALES_ORDER / WORK_ORDER / PURCHASE_ORDER / TRANSFER / ADJUST)
  ref_id               출처 ID
  transfer_id          창고 이동 시 OUT-IN 쌍 연결
  tx_date, created_by

work_order_material    작업지시 투입 자재 목록
  work_order_id, item_id, qty_required, qty_reserved

material_issue         자재 출고 헤더 (작업지시당 1건)
  work_order_id, status (DRAFT / CONFIRMED)

material_issue_item    자재 출고 상세 (LOT 지정)
  material_issue_id, item_id, lot_no, warehouse_id, qty

sales_order_item_lot   수주 품목 출고 이력 (출고 확정 시점에 기록)
  sales_order_item_id, warehouse_id, lot_no, qty
```

### 수불 유형(tx_type) 정의

| 코드 | 설명 | 재고 변화 |
|------|------|----------|
| `PURCHASE_IN` | 구매 입고 | `qty_on_hand++` |
| `MATERIAL_RESERVE` | 작업지시 투입 자재 선점 | `qty_reserved++` |
| `MATERIAL_UNRESERVE` | 작업지시 취소 선점 해제 | `qty_reserved--` |
| `PRODUCTION_OUT` | 자재 출고 확정 (생산 투입) | `qty_on_hand--`, `qty_reserved--` |
| `PRODUCTION_IN` | 생산 완료 입고 | `qty_on_hand++` |
| `SALES_OUT` | 출고 확정 | `qty_on_hand--` |
| `TRANSFER_OUT` | 창고 이동 출고 | `qty_on_hand--` |
| `TRANSFER_IN` | 창고 이동 입고 | `qty_on_hand++` |
| `ADJUST_IN` | 재고 조정 증가 | `qty_on_hand++` |
| `ADJUST_OUT` | 재고 조정 감소 | `qty_on_hand--` |

### 재고 선점 정책

- **선점 대상**: 투입 자재(원자재/반제품) — 완제품 선점 없음
- **선점 시점**: 작업지시 등록 시 (LOT 미지정, 수량만 선점)
- **선점 조건**: 가용 재고(`qty_on_hand - qty_reserved`) >= 필요 수량, 부족 시 작업지시 등록 BLOCK
- **LOT 지정 시점**: 자재 출고 등록 화면에서 담당자가 직접 지정
- **부분 선점 불가**: 전체 수량 선점 가능 여부를 먼저 확인 후 등록

### LOT 관리 정책

- 전 품목 LOT 관리 적용
- 한 작업지시 투입 자재에 여러 LOT 사용 가능 (분산 출고)
- 자재 출고 시 LOT 지정은 담당자가 수동 지정 (자동 FIFO 추천 기능 제공 가능)

### Guard 조건

| 액션 | 조건 |
|------|------|
| 작업지시 등록 | 투입 자재 전체 가용 재고 충분 |
| 생산 시작 | 하위 작업지시 전체 COMPLETED + 자재 출고 전체 CONFIRMED |
| 자재 출고 확정 | LOT 가용 재고 충분 (선점 수량 범위 내) |
| 출고 확정 | 최상위 작업지시 COMPLETED + 완제품 가용 재고 충분 |
| 작업지시 취소 | IN_PROGRESS 이전 상태만 가능 → 선점 자동 해제 |
| 수주 취소 | 연결된 작업지시가 IN_PROGRESS 이상이면 불가 |
| 창고 삭제 | 해당 창고에 재고 존재 시 불가 |

### 작업지시와 BOM 다단계 구조

수주 품목 1건 → 최상위 작업지시 1건 → BOM 구조에 따라 하위 작업지시 N건 생성

```
sales_order_item (1건)
  └→ work_order (최상위, parent_wo_id = null)
       └→ work_order (하위, parent_wo_id = 상위 WO ID)
            └→ material_issue (자재 출고)
```

하위 작업지시 완료 → 반제품 재고 입고 → 상위 작업지시의 자재 출고 확정 가능

## 결과

### 긍정적
- 현재고 O(1) 조회 (스냅샷)
- 완전한 수불 이력 추적 (원장)
- LOT 단위 추적으로 불량/리콜 역추적 가능
- 투입 자재 선점으로 생산 중 자재 부족 방지
- 음수 재고 방지 (Guard 조건)
- 창고 간 이동 이력 추적 (`transfer_id` 연결)

### 부정적 / 트레이드오프
- 스냅샷 + 원장 동시 유지로 모든 재고 변경이 트랜잭션 필요
- 낙관적 락 충돌 시 재시도 로직 필요
- LOT 관리로 인해 입고/출고 화면 복잡도 증가

## 관련 문서
- [개발 로드맵](../plan/development-roadmap.md)
- 관련 ADR: [ADR-001](./001-bom-ui-structure.md), [ADR-002](./002-routing-references-bom.md)
