# 공통코드 × 상태 Enum 혼합 구조 설계

## 배경

상태 코드를 문자열 하드코딩에서 Enum으로 전환하는 과정에서 설계 충돌이 발견됐다.

- **Enum**: 개발자가 코드에 고정 → 컴파일 타임 타입 안전성 보장
- **commoncode**: 사용자가 DB에서 런타임 관리 → 추가/삭제 자유

사용자가 `commoncode`에 새 상태를 추가하면 `SalesOrderStatus.from()` 등에서 예외가 발생하고,
삭제하면 Enum과 DB 간 불일치가 생긴다.

---

## 선택한 방향: C — 혼합 구조

> **Enum = 워크플로우 상태 (시스템 고정)**  
> **commoncode = 표시 라벨 (관리자가 직접 시드, 사용자는 name/sortOrder만 수정)**

워크플로우 로직(`DRAFT → CONFIRMED → ...`)은 비즈니스 로직과 직결되므로
상태 코드 자체는 Enum으로 고정하고, commoncode는 라벨(한글명) 관리 역할로 제한한다.
시스템 상태 코드의 추가/삭제는 개발자(관리자)가 직접 시드 데이터로 관리하며 API로 차단하지 않는다.

---

## 변경 범위

### 1. `CommonCode` 엔티티 — `systemYn` 플래그 추가

```java
// CommonCode.java
@Column(nullable = false)
private boolean systemYn = false;
// true: 시스템 고정 코드 — Enum과 1:1 매핑, 관리자가 시드로 관리
// false: 사용자 정의 코드 — 자유롭게 추가/삭제 가능
```

`systemYn`은 API 보호 목적이 아니라 **프론트엔드에서 시스템 코드임을 식별**하거나
향후 UI에서 삭제 버튼 비활성화 등에 활용할 수 있는 메타데이터 용도다.

### 2. Enum 클래스 — 변경 없음

`SalesOrderStatus`, `QuoteStatus`, `PurchaseOrderStatus` 등 기존 Enum 구조를 그대로 유지한다.

### 3. DB 시드 데이터 — systemYn 구분

상태 Enum이 존재하는 도메인의 코드는 `system_yn = true`로 시드한다.

```sql
INSERT INTO common_code (code_group, code, name, sort_order, system_yn, use_yn) VALUES
  -- 수주 상태
  ('ORDER_STATUS', 'ORDER_STATUS_01', '초안',   1, true, true),
  ('ORDER_STATUS', 'ORDER_STATUS_02', '확정',   2, true, true),
  ('ORDER_STATUS', 'ORDER_STATUS_03', '진행중', 3, true, true),
  ('ORDER_STATUS', 'ORDER_STATUS_04', '완료',   4, true, true),
  ('ORDER_STATUS', 'ORDER_STATUS_05', '취소',   5, true, true),
  -- 견적 상태
  ('QUOTE_STATUS', 'QUOTE_STATUS_01', '초안',     1, true, true),
  ('QUOTE_STATUS', 'QUOTE_STATUS_02', '검토요청', 2, true, true),
  ('QUOTE_STATUS', 'QUOTE_STATUS_03', '승인',     3, true, true),
  ('QUOTE_STATUS', 'QUOTE_STATUS_04', '반려',     4, true, true),
  ('QUOTE_STATUS', 'QUOTE_STATUS_05', '수주전환', 5, true, true),
  -- 발주 상태
  ('PO_STATUS', 'PO_STATUS_01', '초안',     1, true, true),
  ('PO_STATUS', 'PO_STATUS_02', '발주',     2, true, true),
  ('PO_STATUS', 'PO_STATUS_03', '입고완료', 3, true, true),
  ('PO_STATUS', 'PO_STATUS_04', '취소',     4, true, true),
  -- 구매요청 상태
  ('PR_STATUS', 'PR_STATUS_01', '초안',     1, true, true),
  ('PR_STATUS', 'PR_STATUS_02', '검토중',   2, true, true),
  ('PR_STATUS', 'PR_STATUS_03', '승인',     3, true, true),
  ('PR_STATUS', 'PR_STATUS_04', '반려',     4, true, true),
  ('PR_STATUS', 'PR_STATUS_05', '발주완료', 5, true, true),
  -- 출하 상태
  ('SHIPMENT_STATUS', 'SHIPMENT_STATUS_01', '대기',   1, true, true),
  ('SHIPMENT_STATUS', 'SHIPMENT_STATUS_02', '진행중', 2, true, true),
  ('SHIPMENT_STATUS', 'SHIPMENT_STATUS_03', '완료',   3, true, true),
  -- 매출 상태
  ('REVENUE_STATUS', 'REVENUE_STATUS_01', '초안', 1, true, true),
  ('REVENUE_STATUS', 'REVENUE_STATUS_02', '마감', 2, true, true),
  ('REVENUE_STATUS', 'REVENUE_STATUS_03', '취소', 3, true, true),
  -- 입고 상태
  ('GR_STATUS', 'GR_STATUS_01', '초안', 1, true, true),
  ('GR_STATUS', 'GR_STATUS_02', '완료', 2, true, true),
  ('GR_STATUS', 'GR_STATUS_03', '취소', 3, true, true)
ON CONFLICT (code_group, code) DO NOTHING;
```

### 4. 라벨 조회 — QueryRepository에서 common_code JOIN

라벨은 서비스 레이어에서 별도 조회하지 않고, **jOOQ QueryRepository의 SELECT 시 common_code를 JOIN**해서 가져온다.
N+1 없이 한 번의 쿼리로 코드와 라벨을 함께 조회할 수 있다.

**QueryRepository 변경 (salesorder 예시):**
```java
// SalesOrderQueryRepository.java
CommonCode cc = CommonCode.COMMON_CODE;

dsl.select(
    so.ID, so.ORDER_NUMBER, so.STATUS_CODE,
    cc.NAME.as("statusName"),   // 라벨 추가
    ...
)
.from(so)
.leftJoin(cc).on(so.STATUS_CODE.eq(cc.CODE))  // JOIN 추가
...
```

**Response DTO 변경 (salesorder 예시):**
```java
// SalesOrderResponse.java
public record SalesOrderResponse(
    ...
    String statusCode,
    String statusName,   // 추가
    ...
) {
    public static SalesOrderResponse fromRecord(Record r, ...) {
        return new SalesOrderResponse(
            ...
            r.get(so.STATUS_CODE),
            r.get("statusName", String.class),   // 추가
            ...
        );
    }
}
```

**적용 대상 (7개 도메인 동일 패턴):**

| 도메인 | QueryRepository | Response DTO |
|--------|----------------|--------------|
| salesorder | `SalesOrderQueryRepository` | `SalesOrderResponse` |
| quote | `QuoteQueryRepository` | `QuoteResponse` |
| purchaseorder | `PurchaseOrderQueryRepository` | `PurchaseOrderResponse` |
| purchaserequest | `PurchaseRequestQueryRepository` | `PurchaseRequestResponse` |
| shipment | `ShipmentQueryRepository` | `ShipmentResponse` |
| revenue | `RevenueQueryRepository` | `RevenueResponse` |
| goodsreceipt | `GoodsReceiptQueryRepository` | `GoodsReceiptResponse` |

---

## 구조 다이어그램

```
┌──────────────────────────────────────────────────┐
│                commoncode (DB)                   │
│                                                  │
│  systemYn=true          │  systemYn=false        │
│  ─────────────────────  │  ───────────────────   │
│  ORDER_STATUS_01 "초안" │  PRODUCT_TYPE_01       │
│  ORDER_STATUS_02 "확정" │  (사용자 추가/삭제 자유) │
│  ...                    │                        │
│  관리자가 시드로 관리    │                        │
└──────────┬──────────────┴────────────────────────┘
           │ name(라벨) JOIN 조회
           ▼
┌─────────────────────┐
│  SalesOrderStatus   │  워크플로우 규칙 보장
│  Enum (코드 고정)   │  DRAFT → CONFIRMED → ...
└─────────────────────┘
```

---

## 적용 대상 도메인 — 전체 상태값 목록

| 도메인 | Enum | 상태값 |
|--------|------|--------|
| salesorder | `SalesOrderStatus` | DRAFT, CONFIRMED, IN_PROGRESS, COMPLETED, CANCELLED (5) |
| quote | `QuoteStatus` | DRAFT, SUBMITTED, APPROVED, REJECTED, ORDERED (5) |
| purchaseorder | `PurchaseOrderStatus` | DRAFT, ORDERED, RECEIVED, CANCELLED (4) |
| purchaserequest | `PurchaseRequestStatus` | DRAFT, UNDER_REVIEW, APPROVED, REJECTED, ORDERED (5) |
| shipment | `ShipmentStatus` | WAITING, IN_PROGRESS, COMPLETED (3) |
| revenue | `RevenueStatus` | DRAFT, CLOSED, CANCELLED (3) |
| goodsreceipt | `GoodsReceiptStatus` | DRAFT, COMPLETED, CANCELLED (3) |

**총 28개 상태 코드**가 `system_yn = true`로 시드되어야 한다.

---

## 미확인 사항

- [ ] 기존 DB에 commoncode 시드 데이터가 존재하는지 확인
  - 위 SQL은 `ON CONFLICT DO NOTHING` 사용 — `(code_group, code)` unique constraint 필요
