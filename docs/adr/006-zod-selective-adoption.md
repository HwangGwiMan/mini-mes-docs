# ADR-006: 프론트엔드 Zod 선택적 도입

## 상태
승인됨

## 날짜
2026-04-09

## 맥락

프론트엔드는 TypeScript interface로 API DTO를 정의하고, Axios 제네릭(`api.get<InventoryDto[]>`)으로
응답을 캐스팅한다. 이 방식은 컴파일 타임 타입 체크만 보장하며, 런타임에서 백엔드가 예상과 다른
값을 내려줘도 탐지하지 못한다.

주요 문제:

1. **런타임 검증 없음** — TypeScript 타입은 컴파일 후 소거된다. API 응답이 스키마를 실제로
   만족하는지 보장하는 수단이 없다.
2. **`unknown` 타입 unsafe 캐스팅** — `api-error.ts`에서 `err as { response?: { data?: ... } }`
   형태의 캐스팅을 사용해, 런타임 구조가 다를 경우 조용히 `undefined`를 반환한다.
3. **localStorage 읽기** — `useColumnSettings.ts`에서 `localStorage`에 저장된 값을 역직렬화할 때
   스키마 검증 없이 사용한다.

Zod 전면 도입도 검토했으나, 현재 프로젝트는 테스트 러너가 없고, 백엔드와의 API 계약이 비교적
안정적이다. API 응답 DTO마다 interface와 schema를 이중 관리하는 비용 대비 효과가 낮다.

## 검토한 선택지

- **선택지 A: 전면 도입** — 모든 DTO에 Zod schema를 정의하고 API 응답을 `z.parse()`로 검증한다.
  타입 안전성이 극대화되지만, 기존 도메인 파일 전체를 이중으로 유지해야 하는 부담이 크다.
  백엔드 OpenAPI 스펙이 없으면 수동 유지보수 비용이 높다.

- **선택지 B: 미도입** — 현행 TypeScript interface + Axios 제네릭 캐스팅 유지.
  단순하지만 런타임 경계(unknown 파싱, localStorage)에서의 취약점을 해소하지 못한다.

- **선택지 C: 선택적 도입** — 런타임 안전성이 실질적으로 필요한 경계에만 Zod를 적용한다.
  API 응답 DTO는 현행 방식을 유지한다.

## 결정

**선택지 C — 선택적 도입**을 채택한다.

### 적용 대상

| 대상 | 이유 |
|---|---|
| `src/types/api-error.ts` | `unknown` 타입 파싱 — `z.safeParse()`로 대체하면 unsafe 캐스팅 제거 가능 |
| `src/composables/useColumnSettings.ts` | localStorage 역직렬화 — 외부 데이터이므로 스키마 검증이 의미 있음 |
| 복잡한 폼 유효성 검사 | 커스텀 모달 등에서 필요 시 그때그때 추가 |

### 적용하지 않는 대상

| 대상 | 이유 |
|---|---|
| API 응답 DTO (`src/api/*.ts`) | 백엔드 계약이 안정적이고, 테스트 없는 환경에서 이중 관리 비용이 효과를 상회 |
| 단순 Request shape | `TransferRequest`, `AdjustRequest` 등 내부에서만 생성하는 객체 |

### 향후 전환 조건

백엔드가 OpenAPI 스펙을 제공하게 되면 `@hey-api/openapi-ts` + Zod 자동 생성 방식으로
전면 전환을 재검토한다. 수동 이중 관리 없이 schema를 유지할 수 있을 때 전면 도입이 합리적이다.

## 결과

### 긍정적
- 런타임 경계(`unknown`, localStorage)에서 실질적인 안전성 확보
- 기존 DTO 파일 변경 없음 — 마이그레이션 비용 최소화
- 필요한 곳에 점진적으로 확장 가능

### 부정적 / 트레이드오프
- API 응답 DTO는 여전히 런타임 검증 없음 (컴파일 타임 타입 체크에만 의존)
- Zod 적용 범위가 파일마다 다르므로, 새 기여자가 일관성 없다고 느낄 수 있음

## 관련 문서
- 관련 ADR: 없음
