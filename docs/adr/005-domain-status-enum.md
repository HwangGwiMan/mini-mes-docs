# ADR-005: 도메인 상태 코드를 Enum으로 관리

## 상태
<!-- 제안됨 | 검토중 | 승인됨 | 거부됨 | 폐기됨 | 대체됨 -->
승인됨

## 날짜
2026-03-31

## 맥락

현재 7개 도메인(quote, salesorder, shipment, revenue, purchaseorder, purchaserequest, goodsreceipt)에서
상태 코드를 `String statusCode` 필드로 관리하며, `"QUOTE_STATUS_01"` 같은 문자열 리터럴을
엔티티·서비스·QueryRepository 전반에 직접 하드코딩하고 있다.

주요 문제:

1. **타입 안전성 없음** — 오타(`"PO_STATUS_O1"`)가 있어도 컴파일 오류가 발생하지 않는다.
2. **일관성 없음** — `ShipmentService`, `RevenueService`만 일부 상수를 정의하고,
   나머지 도메인은 리터럴을 그대로 사용한다.
3. **도메인 경계 침범** — `SalesOrderService`가 `"QUOTE_STATUS_03"`을, `PurchaseOrderService`가
   `"PR_STATUS_03"`을 직접 참조해, 변경 시 파급 범위를 추적하기 어렵다.
4. **IDE 지원 없음** — 자동완성·참조 찾기·리팩터링이 불가하다.

모든 상태 코드는 `CommonCode` 시스템(DB)에도 등록되어 있어, Enum 도입 후에도
코드 값 자체(`"QUOTE_STATUS_01"` 등)는 DB와 동일하게 유지해야 일관성을 보장할 수 있다.

## 검토한 선택지

- **선택지 A: 각 도메인 패키지 내 개별 Enum 배치**
  예: `quote/domain/QuoteStatus.java`, `salesorder/domain/SalesOrderStatus.java`
  도메인 응집도를 유지하며, 각 Enum이 해당 도메인 내부에서만 사용된다.

- **선택지 B: 공통 패키지에 모든 상태 Enum 집중**
  예: `common/domain/status/QuoteStatus.java`
  한 곳에서 일괄 관리되지만, 도메인 경계가 흐려지고 공통 패키지가 비대해진다.

- **선택지 C: 현행 유지 (문자열 상수화만 개선)**
  서비스 클래스에 `private static final String STATUS_DRAFT = "..."` 상수를 추가해
  리터럴 직접 사용을 줄인다. 타입 안전성은 여전히 없다.

## 결정

**선택지 A — 각 도메인 패키지 내 개별 Enum 배치**를 채택한다.

### Enum 설계 원칙

1. **코드 값 보존** — `code` 필드에 기존 CommonCode 코드 값을 그대로 저장한다.
   DB 데이터 마이그레이션 없이 동작해야 하므로 `@Enumerated(EnumType.STRING)` 대신
   JPA `AttributeConverter`를 사용해 `code` ↔ Enum 변환을 담당한다.

2. **배치 위치** — `{domain}/domain/{DomainName}Status.java`

3. **도메인 간 참조 제거** — 다른 도메인의 상태를 직접 문자열로 참조하던 코드는
   해당 도메인 Enum의 `.code()` 호출로 교체한다.
   예: `salesorder`가 `"QUOTE_STATUS_03"`을 참조 → `QuoteStatus.APPROVED.code()`

### 적용 대상 도메인 및 Enum 멤버

| 도메인 | Enum 클래스 | 멤버 (코드값) |
|--------|------------|--------------|
| quote | `QuoteStatus` | `DRAFT("QUOTE_STATUS_01")`, `SUBMITTED("QUOTE_STATUS_02")`, `APPROVED("QUOTE_STATUS_03")`, `REJECTED("QUOTE_STATUS_04")`, `ORDERED("QUOTE_STATUS_05")` |
| salesorder | `SalesOrderStatus` | `DRAFT("ORDER_STATUS_01")`, `CONFIRMED("ORDER_STATUS_02")`, `IN_PROGRESS("ORDER_STATUS_03")`, `COMPLETED("ORDER_STATUS_04")`, `CANCELLED("ORDER_STATUS_05")` |
| shipment | `ShipmentStatus` | `WAITING("SHIPMENT_STATUS_01")`, `IN_PROGRESS("SHIPMENT_STATUS_02")`, `COMPLETED("SHIPMENT_STATUS_03")` |
| revenue | `RevenueStatus` | `DRAFT("REVENUE_STATUS_01")`, `CLOSED("REVENUE_STATUS_02")`, `CANCELLED("REVENUE_STATUS_03")` |
| purchaseorder | `PurchaseOrderStatus` | `DRAFT("PO_STATUS_01")`, `ORDERED("PO_STATUS_02")`, `RECEIVED("PO_STATUS_03")`, `CANCELLED("PO_STATUS_04")` |
| purchaserequest | `PurchaseRequestStatus` | `DRAFT("PR_STATUS_01")`, `UNDER_REVIEW("PR_STATUS_02")`, `APPROVED("PR_STATUS_03")`, `REJECTED("PR_STATUS_04")`, `ORDERED("PR_STATUS_05")` |
| goodsreceipt | `GoodsReceiptStatus` | `DRAFT("GR_STATUS_01")`, `COMPLETED("GR_STATUS_02")`, `CANCELLED("GR_STATUS_03")` |

### JPA 변환 방식

각 Enum마다 `AttributeConverter`를 내부 정적 클래스로 구현한다.

```java
// 예: QuoteStatus
public enum QuoteStatus {
    DRAFT("QUOTE_STATUS_01"),
    SUBMITTED("QUOTE_STATUS_02"),
    APPROVED("QUOTE_STATUS_03"),
    REJECTED("QUOTE_STATUS_04"),
    ORDERED("QUOTE_STATUS_05");

    private final String code;

    QuoteStatus(String code) { this.code = code; }

    public String code() { return code; }

    public static QuoteStatus from(String code) {
        for (QuoteStatus s : values()) {
            if (s.code.equals(code)) return s;
        }
        throw new IllegalArgumentException("Unknown QuoteStatus code: " + code);
    }

    @Converter(autoApply = true)
    public static class JpaConverter implements AttributeConverter<QuoteStatus, String> {
        @Override public String convertToDatabaseColumn(QuoteStatus s) { return s == null ? null : s.code(); }
        @Override public QuoteStatus convertToEntityAttribute(String code) { return code == null ? null : QuoteStatus.from(code); }
    }
}
```

엔티티 필드는 `String statusCode` → `QuoteStatus status`로 변경하며,
`@Column(name = "status_code")` 매핑은 유지한다.

## 결과

### 긍정적
- 컴파일 타임 타입 안전성 확보 — 잘못된 상태 값 사용 시 즉시 오류
- IDE 자동완성 및 참조 추적 가능
- 도메인 간 문자열 직접 참조 제거 → 변경 영향 범위 명확화
- 서비스 내 `private static final String` 상수 제거로 코드 단순화
- DB 스키마 변경 없음 (코드 값 그대로 유지)

### 부정적 / 트레이드오프
- `AttributeConverter`를 7개 Enum에 각각 작성해야 함 (반복 구조)
- jOOQ QueryRepository에서 `.eq(status.code())` 형태로 명시적 변환 필요
- CommonCode DB 데이터와 Enum 멤버를 동기화하는 것은 개발자 책임으로 남음

## 관련 문서
- 관련 ADR: [ADR-004](./004-inventory-transaction-design.md)
