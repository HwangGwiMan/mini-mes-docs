# ADR-010: 메뉴 접근 권한 — roles 필드 + 프론트엔드 필터링

## 상태
승인됨

## 날짜
2026-04-22

## 맥락

사이드바 메뉴는 현재 로그인한 사용자라면 누구나 동일하게 보인다. 장기적으로 사용자 역할(role)에 따라 접근 가능한 메뉴를 제한해야 하는 요구사항이 있다. 예를 들어 시스템 설정 메뉴는 관리자만, 구매 관련 메뉴는 구매 담당자만 접근하도록 제어가 필요하다.

메뉴 접근 제어 구현 방식으로 두 가지 방향을 검토했다:

1. **메뉴 정의를 DB에 저장하고 API로 조회**하는 방식
2. **코드 내 정적 메뉴 정의에 roles 필드를 추가**하고 프론트엔드에서 필터링하는 방식

## 검토한 선택지

- **선택지 A — DB 저장 + API 조회**:
  메뉴 목록을 DB 테이블로 관리하고 로그인 후 사용자 권한에 맞는 메뉴를 서버에서 내려준다.
  - 문제 1: 메뉴의 `icon`이 Vue 컴포넌트 객체(`lucide-vue-next`)이므로 DB에 저장할 수 없다. 결국 프론트엔드에 아이콘 매핑 테이블(`{ "Database": Database, ... }`)이 남아야 한다.
  - 문제 2: 메뉴의 `path`는 Vue Router에 등록된 라우트와 1:1 대응한다. 라우트는 코드 배포 없이 추가할 수 없으므로, 메뉴를 DB에서 동적으로 관리해도 실질적인 유연성이 없다.
  - 문제 3: MES 특성상 메뉴 변경은 항상 새 화면 개발(코드 배포)과 함께 발생한다. DB 관리의 운영 편의성 이점이 없다.

- **선택지 B — 정적 menus.ts + roles 필드 + 프론트엔드 필터링** (채택):
  `menus.ts`의 `MenuLeaf`와 `MenuGroup` 인터페이스에 `roles?: string[]` 필드를 추가하고, `AppSidebar.vue`의 `visibleMenus` computed에서 현재 사용자의 role과 대조해 필터링한다.
  - `roles`가 미지정이면 전체 공개(하위 호환 유지).
  - 그룹의 모든 자식이 숨겨지면 그룹 자체도 사라진다.
  - 사용자 role은 로그인 후 `/graphql { me { role } }`에서 받아 Pinia auth 스토어에 저장한다.

## 결정

**선택지 B**를 채택한다.

### 변경 범위

| 파일 | 변경 내용 |
|------|-----------|
| `src/config/menus.ts` | `MenuLeaf`, `MenuGroup`에 `roles?: string[]` 추가 |
| `src/stores/auth.ts` | `role` ref, `setRole()` 메서드 추가. `logout()` 시 role도 초기화 |
| `src/layouts/AppSidebar.vue` | `visibleMenus` computed에 `hasAccess()` 필터 적용 |

### roles 미지정 시 동작

`roles` 필드를 지정하지 않은 메뉴는 기존과 동일하게 전체 공개된다. 현재는 모든 메뉴에 `roles`를 지정하지 않았으므로 동작 변화 없음.

### role 저장 흐름

```
로그인 성공
  → useScreenInit().initialize() 호출 (GraphQL /graphql { me { role } })
  → authStore.setRole(role) 호출
  → localStorage('role') 저장 (새로고침 후에도 유지)
  → visibleMenus computed 재계산 → 사이드바 갱신
```

### 주의: 이 방식은 UI 필터링이며 라우터 가드가 아님

메뉴에서 접근 불가 항목을 숨기는 것은 UX 처리다. URL 직접 입력을 통한 우회 접근을 막으려면 별도로 Vue Router의 `beforeEach` 가드에서 role 검사를 추가해야 한다. 이는 후속 작업으로 남겨둔다.

## 결과

### 긍정적
- 코드 배포 없이 `menus.ts`에 `roles` 필드만 추가하면 메뉴별 접근 제어 적용 가능
- DB/API 추가 없이 기존 GraphQL `me` 쿼리 재활용
- `roles` 미지정 메뉴는 전체 공개로 유지되어 기존 화면에 영향 없음
- 그룹 내 모든 자식이 숨겨지면 그룹 헤더도 자동 제거 — 빈 그룹이 노출되지 않음

### 부정적 / 트레이드오프
- URL 직접 입력 시 라우터 가드가 없으면 우회 가능 (후속 작업 필요)
- 역할 목록이 코드에 문자열 리터럴(`'ADMIN'`, `'MANAGER'` 등)로 분산되므로 role 이름 변경 시 menus.ts와 백엔드를 함께 수정해야 함
- 메뉴를 DB에서 동적으로 관리하는 요구사항(관리자 UI에서 메뉴 권한 설정 등)이 생기면 이 방식으로 대응 불가 — 그 시점에 재검토 필요

## 관련 문서
- 없음
