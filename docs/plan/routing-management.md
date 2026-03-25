# 라우팅(공정순서) 관리 설계

## 개요

라우팅(Routing)은 특정 BOM에 대해 **어떤 공정을 어떤 순서로 거쳐야 하는지**를 정의한 기준 정보다.
BOM이 "무엇으로 만드는가"를 정의한다면, 라우팅은 "어떻게 만드는가"를 정의한다.

BOM 1개에 라우팅 1개가 대응된다. BOM이 버전을 가지므로, 라우팅은 별도 버전을 관리하지 않는다.

---

## 데이터 구조

```
Routing (헤더)
  ├── BOM                      ← 어떤 BOM의 라우팅인가 (1:1)
  └── 상태 (활성/비활성)

RoutingStep (공정 단계)
  ├── 공정 (Process)           ← 어떤 공정인가
  ├── 단계 순서 (stepOrder)    ← 몇 번째 공정인가
  ├── 표준 작업 시간 (분)      ← 해당 BOM 기준 소요 시간 (공정 기본값과 다를 수 있음)
  └── 비고
```

---

## 기능 목록

### 1. 라우팅 목록 조회
- 완제품 코드/명, BOM 버전, 활성여부로 검색
- 결과: 완제품코드, 완제품명, BOM 버전, 공정 수, 활성여부

### 2. 라우팅 상세 조회
- 헤더 정보(연결 BOM 포함) + 공정 단계 목록 표시
- 동일 품목의 다른 버전 BOM 라우팅 목록 확인 가능

### 3. 라우팅 등록
- BOM 선택 (활성 BOM 중 아직 라우팅이 없는 것만 선택 가능)
- 공정 단계 동적 추가: 공정 선택, 순서 지정, 표준시간 입력
- 검증: 동일 BOM에 대한 중복 등록 방지, 같은 라우팅 내 동일 공정 중복 방지

### 4. 라우팅 수정
- 공정 단계 목록 수정 (연결 BOM은 수정 불가)
- 단계 전체를 교체하는 방식 (clearSteps → addSteps)

### 5. 라우팅 비활성화
- 삭제 대신 `activeYn = false` 처리로 이력 보존

---

## 비즈니스 규칙

| 규칙 | 처리 |
|------|------|
| BOM 1개에 라우팅 1개만 허용 | 409 Conflict |
| BOM 존재 여부 검증 | `BomService.exists()` |
| 공정 존재 여부 검증 | `ProcessService.exists()` |
| 동일 라우팅 내 동일 공정 중복 입력 금지 | 400 Bad Request |
| 삭제 없음 — 비활성 처리만 허용 | `activeYn = false` |

---

## 백엔드 설계

### 패키지 구조

```
routing/
  api/
    RoutingController.java
    dto/
      RoutingCreateRequest.java
      RoutingUpdateRequest.java
      RoutingStepRequest.java
      RoutingResponse.java       ← 목록/상세 공용 (steps 필드는 상세 시 포함)
  application/
    RoutingService.java
    package-info.java            ← @NamedInterface
  domain/
    Routing.java
    RoutingStep.java
    RoutingRepository.java
  internal/
    RoutingQueryRepository.java  ← jOOQ 조회
  package-info.java              ← @ApplicationModule
```

### 엔티티

#### `Routing` (헤더)

| 필드 | 타입 | 제약 |
|------|------|------|
| id | Long | PK, auto |
| bomId | Long | Bom FK, UNIQUE (NOT NULL) |
| activeYn | Boolean | 기본 true |
| + BaseEntity | | createdAt, updatedAt, createdBy, updatedBy, @Version |

유니크 제약: `bomId` (BOM당 라우팅 1개)

#### `RoutingStep` (공정 단계)

| 필드 | 타입 | 제약 |
|------|------|------|
| id | Long | PK, auto |
| routing | Routing | ManyToOne (NOT NULL) |
| processId | Long | Process FK (NOT NULL) |
| stepOrder | int | 단계 순서 (NOT NULL) |
| standardTime | Integer | 표준 작업 시간(분), nullable |
| remarks | String(200) | nullable |
| + BaseEntity | | |

### REST API

| Method | Path | 설명 |
|--------|------|------|
| GET | `/api/routings` | 목록 조회 (itemCode, itemName, bomVersion, activeYn) |
| GET | `/api/routings/{id}` | 단건 상세 조회 |
| GET | `/api/routings/by-bom/{bomId}` | BOM별 라우팅 조회 (0 또는 1건) |
| GET | `/api/routings/by-item/{itemId}` | 품목별 라우팅 목록 (버전 이력 탐색용) |
| POST | `/api/routings` | 라우팅 등록 |
| PUT | `/api/routings/{id}` | 라우팅 수정 |
| PATCH | `/api/routings/{id}/deactivate` | 비활성 처리 |

### DTO

**RoutingResponse** (목록용): id, bomId, itemId, itemCode, itemName, bomVersion, activeYn, stepCount

**RoutingDetailResponse** (상세용): RoutingResponse 필드 + steps(RoutingStepResponse[])

**RoutingStepResponse**: id, processId, processCode, processName, stepOrder, standardTime, remarks

**RoutingCreateRequest**: bomId, steps[]

**RoutingUpdateRequest**: steps[]

**RoutingStepRequest**: processId, stepOrder, standardTime, remarks

### 모듈 의존성

```java
@ApplicationModule(allowedDependencies = {
    "bom::application",
    "process::application",
    "common::domain", "common::exception", "common::util",
    "jooq::tables"
})
package com.github.gwiman.mini_mes_backend.routing;
```

> `item` 정보(코드, 명)는 jOOQ 조회 시 BOM → Item 조인으로 가져오므로 `item::application` 직접 의존 불필요

---

## 프론트엔드 설계

### API 레이어 (`frontend/src/api/routing.ts`)

```typescript
export interface RoutingDto {
  id: number
  bomId: number
  itemId: number
  itemCode: string
  itemName: string
  bomVersion: string
  activeYn: boolean
  stepCount: number
}

export interface RoutingDetailDto extends RoutingDto {
  steps: RoutingStepDto[]
}

export interface RoutingStepDto {
  id: number
  processId: number
  processCode: string
  processName: string
  stepOrder: number
  standardTime?: number
  remarks?: string
}

export interface RoutingCreateRequest {
  bomId: number
  steps: RoutingStepRequest[]
}

export interface RoutingUpdateRequest {
  steps: RoutingStepRequest[]
}

export interface RoutingStepRequest {
  processId: number
  stepOrder: number
  standardTime?: number
  remarks?: string
}
```

### 라우터

```
/routing          → RoutingListView    (목록/검색)
/routing/:id      → RoutingDetailView  (상세)
```

### 화면 1 — `RoutingListView.vue`

- 검색: 완제품 코드(text), 완제품명(text), BOM 버전(text), 활성여부(select)
- 테이블 컬럼: 완제품명, 완제품코드, BOM 버전, 공정 수, 활성여부, 상세 버튼
- 등록 버튼 → `RoutingFormModal` 열기

### 화면 2 — `RoutingDetailView.vue`

- 헤더: 완제품명, BOM 버전, 활성여부, 연결 BOM 링크(`/bom/:bomId`)
- 액션: 수정(`RoutingFormModal`), 비활성화
- 버전 이력: `routingApi.getByItemId(itemId)` → 동일 품목의 다른 버전 라우팅 클릭 시 해당 상세로 이동
- 공정 단계 테이블: 순서, 공정코드, 공정명, 표준시간, 비고

### `RoutingFormModal.vue`

- 등록 모드: BOM 선택 드롭다운 (라우팅 미등록 BOM 목록)
- 수정 모드: BOM 표시만 (변경 불가)
- 공정 단계: 공정 선택 드롭다운 + 표준시간 + 비고, 행 추가/삭제/순서 변경

---

## 개발 순서

1. **백엔드**: `Routing` + `RoutingStep` 엔티티 → Repository → Service → Controller → DTO
2. **DB**: JPA `ddl-auto: update` 로 자동 생성
3. **jOOQ 재생성**: `./gradlew jooqCodegen`
4. **프론트엔드**: `api/routing.ts` → 라우터 → `RoutingListView` → `RoutingFormModal` → `RoutingDetailView`
5. **사이드바 메뉴 등록**

---

## 참조 파일

| 역할 | 파일 경로 |
|------|----------|
| 유사 백엔드 (헤더+라인) | `backend/.../bom/` |
| 공정 마스터 | `backend/.../process/` |
| 유사 프론트 복합 폼 | `frontend/src/views/BomListView.vue`, `BomDetailView.vue` |
| BOM 설계 문서 | `docs/plan/bom-management.md` |
