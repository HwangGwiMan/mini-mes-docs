# ADR-011: 화면 초기화 시 공통코드 조회를 GraphQL 단일 요청으로 통합

- **상태**: 승인됨
- **날짜**: 2026-04-24
- **관련 파일**: `frontend/src/composables/useScreenInit.ts`, `backend/src/main/resources/graphql/schema.graphqls`

---

## 배경

각 CRUD 화면의 `onMounted`에서 두 가지 요청이 순차 또는 병렬로 발생한다:

1. GraphQL `{ me { username role employeeId } }` — 현재 로그인 사용자 조회
2. REST `POST /api/common-codes/search` — 셀렉트 박스용 공통코드 조회 (화면마다 1~N회)

ItemView 기준으로 `ITEM_TYPE`, `UNIT` 두 그룹을 가져오므로 총 3번의 HTTP 요청이 발생한다.
화면이 늘어날수록 공통코드 그룹 수만큼 요청이 누적된다.

## 결정

GraphQL의 **필드 alias** 기능을 활용해 `me` 조회와 여러 공통코드 그룹 조회를 **단일 GraphQL 요청으로 통합**한다.

### 쿼리 형태

그룹코드 이름을 alias로 사용해 동적으로 쿼리를 구성한다:

```graphql
{
  me { username role employeeId }
  ITEM_TYPE: commonCodes(groupCode: "ITEM_TYPE") { code name }
  UNIT: commonCodes(groupCode: "UNIT") { code name }
}
```

응답 구조:
```json
{
  "data": {
    "me": { "username": "...", "role": "...", "employeeId": 1 },
    "ITEM_TYPE": [{ "code": "ITEM_TYPE_01", "name": "원자재" }, ...],
    "UNIT": [{ "code": "UNIT_01", "name": "EA" }, ...]
  }
}
```

### 프론트엔드 API 변경

`useScreenInit.initialize()`의 시그니처를 확장한다:

```ts
// 변경 전
async function initialize(): Promise<CurrentUser>

// 변경 후
async function initialize(groupCodes?: string[]): Promise<{
  currentUser: CurrentUser
  codes: Record<string, { value: string; label: string }[]>
}>
```

내부적으로 `groupCodes` 배열을 순회해 alias 필드를 동적으로 생성하고,
응답에서 `me`를 제외한 나머지 키를 `codes` 맵으로 반환한다.

### 호출 예시 (ItemView)

```ts
// 변경 전
await initialize()
const [itemTypeRes, unitRes] = await Promise.all([
  commonCodeApi.search('ITEM_TYPE'),
  commonCodeApi.search('UNIT'),
])
itemTypeOptions.value = itemTypeRes.data.map(c => ({ value: c.code, label: c.name }))
unitOptions.value = unitRes.data.map(c => ({ value: c.code, label: c.name }))

// 변경 후
const { codes } = await initialize(['ITEM_TYPE', 'UNIT'])
itemTypeOptions.value = codes['ITEM_TYPE']
unitOptions.value = codes['UNIT']
```

### 백엔드 변경

`schema.graphqls`에 `commonCodes` 쿼리와 `CommonCodeOption` 타입을 추가한다:

```graphql
type Query {
  me: CurrentUser
  commonCodes(groupCode: String!): [CommonCodeOption!]!
}

type CommonCodeOption {
  code: String!
  name: String!
}
```

`CommonCodeGraphqlController`(신규)를 추가해 기존 `CommonCodeService.findByGroup()`을 위임 호출한다.

## 검토한 대안

### 대안 A: REST 병렬 요청 유지
현재 방식. 화면마다 `Promise.all`로 묶어 병렬 처리하고 있으나,
요청 수 자체를 줄이지 못하고 `onMounted` 코드가 화면마다 반복된다.

### 대안 B: 공통코드 전체를 한 번에 캐싱
앱 시작 시 모든 공통코드 그룹을 Pinia store에 적재하는 방식.
초기 로딩이 무거워지고, 공통코드가 많아질수록 불필요한 데이터를 미리 받아야 한다.
화면별 필요 그룹만 선택적으로 가져오는 현재 설계 의도와 맞지 않는다.

### 대안 C: GraphQL `commonCodesByGroups(groupCodes: [String!]!)` 단일 필드
여러 그룹을 배열로 받아 `[{ groupCode, items }]` 형태로 반환하는 방식.
alias 방식보다 스키마가 명시적이지만, 응답에서 `groupCode`로 다시 인덱싱하는 코드가 필요하고
alias 방식 대비 이점이 없어 채택하지 않았다.

## 결과

- HTTP 요청 수: 화면 진입 시 `2 + N`회 → **1회**로 감소
- `onMounted` 코드량 감소 및 화면 간 패턴 일관성 확보
- 공통코드 REST 엔드포인트(`/api/common-codes/search`)는 공통코드 관리 화면 자체의 CRUD용으로 유지

## 제약

- GraphQL alias는 식별자 규칙(영문자·숫자·언더스코어, 숫자 시작 불가)을 따른다.
  현재 공통코드 그룹코드 생성 규칙(`CODEGROUP_NN`)은 이 규칙을 만족하므로 문제없다.
  향후 그룹코드 명명 규칙 변경 시 alias 유효성을 함께 검토해야 한다.
