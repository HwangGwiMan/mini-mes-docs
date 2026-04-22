# ADR-009: 폼 모달 공통화 — ModalShell / LineItemTable / useFormModal

## 상태
승인됨

## 날짜
2026-04-22

## 맥락

프론트엔드에 17개의 모달 컴포넌트가 있었고, 그 중 헤더+라인 테이블 구조를 갖는 도메인 전용 모달 8개(QuoteFormModal, OrderFormModal, PurchaseRequestFormModal, PurchaseOrderFormModal, GoodsReceiptFormModal, WorkOrderFormModal, ShipmentFormModal, RevenueFormModal)에 동일한 코드가 반복되고 있었다.

반복 범위:
- `<Teleport>` + `<Transition>` + 오버레이/백드롭 구조: 8개 파일
- 헤더(타이틀+X버튼) / 에러블록(AlertCircle) / 푸터(취소+저장+Loader2 스피너): 8개 파일
- `<style scoped>` 모달 트랜지션 애니메이션: 8개 파일
- `watch(modelValue)` → 폼 초기화 + `internalError` ref + `handleSubmit` 유효성 패턴: 6개 파일
- 라인 테이블 섹션(행 추가 버튼 + `<table>` 래퍼 + 빈 상태 메시지): 6개 파일

총 약 400줄의 중복 코드가 존재했다. 새 도메인 모달 추가 시 동일 코드를 복사해야 했고, 스타일·UX 변경 시 모든 파일을 일괄 수정해야 하는 유지보수 부담이 있었다.

## 검토한 선택지

- **선택지 A — CrudModal 래핑**: 기존 `CrudModal`을 확장해 라인 테이블까지 지원하도록 만든다.
  - 문제: `fields: FieldDef[]` → `Record<string, string>` 인터페이스가 라인 테이블의 동적 행 추가/삭제, 품목 select → 금액 자동계산, 행별 유효성 검사와 근본적으로 맞지 않는다. 복잡도가 CrudModal 내부로 이동할 뿐이다.

- **선택지 B — 공통 서브컴포넌트 + 컴포저블 분리** (채택):
  - `ModalShell.vue`: 오버레이/헤더/에러/푸터 껍데기
  - `LineItemTable.vue`: 라인 테이블 래퍼
  - `useFormModal()`: watch/internalError/handleSubmit 공통 로직
  - 각 전용 모달은 도메인 고유 필드와 라인 컬럼 정의에만 집중한다.

- **선택지 C — 현상 유지**: 반복 코드를 그대로 둔다.
  - 단기적으로 안전하지만 모달 수가 늘어날수록 부담이 가중된다.

## 결정

**선택지 B**를 채택한다.

3개의 공통 요소를 신규 생성하고, 기존 8개 모달을 이를 활용하도록 리팩토링한다.

### 컴포넌트/컴포저블 역할 분담

| 요소 | 위치 | 역할 |
|------|------|------|
| `ModalShell.vue` | `src/components/` | Teleport/오버레이/헤더/에러블록/푸터. `locked` prop으로 저장버튼 숨김 및 취소→닫기 전환. `status-badge` 슬롯으로 도메인별 상태 배지 삽입 |
| `LineItemTable.vue` | `src/components/` | 라인 섹션 헤더(라벨+"행 추가" 버튼) + `<table>` 래퍼. `head`/`body`/`footer` 슬롯 제공. 행 삭제 버튼은 각 모달의 `body` 슬롯 내에서 직접 렌더링 |
| `useFormModal<TDto, TReq>()` | `src/composables/` | `watch(modelValue)` → `onOpen` 콜백 호출, `internalError` ref 관리, `handleSubmit` (잠금 확인 → `validate()` → `buildRequest()` → `onConfirm()`) |

### CrudModal과의 관계

`CrudModal`은 라인 테이블이 없는 단순 CRUD 화면(Partner, Item, Process 등)에서 그대로 유지한다. `ModalShell`은 `CrudModal`을 대체하지 않으며, 복잡한 도메인 전용 모달의 공통 껍데기로만 사용한다.

### 범위에서 제외한 모달

- `BomFormModal`, `RoutingFormModal`: `onMounted`에서 직접 API를 호출하고 `emit('saved')`를 사용하는 독자적 패턴. 별도 후속 작업으로 처리.
- `ShipmentCompleteModal`, `InventoryTransferModal`, `InventoryAdjustModal`: 라인 테이블 없거나 구조가 단순해 필요 시 선택적으로 적용.

### WorkOrderFormModal 특이사항

`useFormModal`의 `onOpen`은 동기 함수이나 WorkOrderFormModal은 품목 선택 시 BOM 목록을 비동기로 로드해야 한다. 이 경우 `watch(modelValue)`를 직접 유지하고 `internalError`와 `handleSubmit`만 수동으로 구현한다.

## 결과

### 긍정적
- 반복 코드 약 400줄 제거
- 오버레이/에러/푸터 UI를 한 곳(`ModalShell`)에서 관리 — 스타일 변경 시 단일 파일만 수정
- 새 도메인 모달 작성 시 껍데기 없이 도메인 필드와 라인 컬럼만 작성
- 상태 잠금·에러 표시 동작이 전체 모달에서 일관됨

### 부정적 / 트레이드오프
- `ModalShell`의 슬롯 구조를 이해해야 새 모달을 작성할 수 있어 초기 학습 비용이 소폭 증가
- `LineItemTable`의 `body` 슬롯 안에서 행 삭제 버튼을 각 모달이 직접 렌더링해야 함 (컬럼 구조가 모달마다 달라 완전한 추상화 불가)
- `useFormModal`을 사용하지 못하는 비동기 초기화 케이스(WorkOrderFormModal)에서는 `watch`를 직접 작성해야 함

## 관련 문서
- 없음
